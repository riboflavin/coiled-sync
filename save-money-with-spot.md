---
title: Save Money with Spot
description: The cloud is wonderful but expensive. Spot/preemptible instances offer dramatic cost savings, but using them well requires considerable nuance...
blogpost: true
date: 
author: 
---

# Save Money with Spot

The cloud is wonderful but expensive.  

Spot/preemptible instances offer dramatic cost savings (2–3x), but only if you can…

1. Find enough in your region
2. Handle their sudden disappearance

Doing this well requires considerable nuance which we explore in this post. 

We optimized this process while making Coiled, a cloud SaaS product around deploying Python distributed computing with Dask. We'll use Coiled/Dask workloads as a running example, but these lessons should apply broadly.

## Spot is Cheap

Both AWS and Google Cloud let you buy idle compute resources at a significant discount. For example, here are the costs for a 2vCPU 8GiB machine on each cloud:

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/63b73b6da9392c4a847b8558_Spot%20vs%20OnDeman%20Chart.png" loading="lazy" alt="">

Spot instances are about a quarter the cost… so what's the catch?

Because spot instances are excess capacity, **they can be harder to get and harder to keep**. Example issues:

- You might request 100 spot workers and only get 80
- You might get 100, but after your cluster is up and running you might lose some

For some workloads, this is fine. For others, less so. Which raises the question:

*How can we get as much spot as possible, while also delivering a consistent user experience?*

After iterating on this problem with our users, we've come up with an approach that we've found to work well:

- Select the best zone for getting spot instances
- Use on-demand instances when spot is unavailable
- Automatically replace spot instances with changes in cloud provider availability

When you spin up a cluster using Coiled, you can now enable these features with [these keyword arguments](https://docs.coiled.io/user_guide/costs/spot.html):

<pre class="language-python"><code class="language-python hljs"><span class="hljs-keyword">import</span> coiled

cluster = coiled.Cluster(
    use_best_zone=<span class="hljs-literal">True</span>, 
    spot_policy=<span class="hljs-string">"spot_with_fallback"</span>,
    ...,
)
</code></pre>

## 1 – Choose the right availability zone 

Before we worry about spot/preemptible instances going away, first we need to find enough spot. To do this well, we want to choose the data center or availability zone with the most excess capacity.

Clouds (AWS, GCP, Azure) have lots of data centers. These data centers are grouped into "**zones**", and one or more zones make up a "**region**". Usually you care about what **region** you use, because the sorts of workloads that people run on Coiled usually involve processing lots of data, and moving data between **regions** costs money.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/63b744967f3626915157c631_Spot%20Savings%20-%20Cloud%20Diagram.png" loading="lazy" alt="">

You should also care that all of your instances are in a **single zone**, because moving data between zones also costs money, but usually you don't need to care which **specific zone** you use. Data is typically stored in a region and not a specific zone within that region.

Different availability zones have different capacity. This applies to Spot, and also other rare instance types like GPUs and TPUs. By being flexible to the right availability zone we can often get more of the thing we want, while still being local to our data.

### Example

We applied this approach to internal workloads, large scale Dask benchmarks run by our in-house OSS teams. These regular jobs spin up many hundreds of workers for benchmarking Dask performance at scale. In the first two weeks of December, we spun up **over 52000** m6i.large spot workers. Out of all those requests for workers, there were exactly **60** times when we failed to get the spot instance but would have been able to get the worker as an on-demand instance.

As availability shifted, you can see that these workers came from different AWS availability zones:

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/63b607ac0681a7842c28c666_image3.png" loading="lazy" alt="">

Prior to this feature, we didn't even use Spot. Consistency in these benchmarks was important enough to our internal teams, and the experience was poor enough that we just paid the higher bill. *Since implementing this feature (and those below) we've heard no complaints from engineering, and nothing but praise from finance*.

## 2 – Spot with Fallback to On-Demand

Even when choosing the best zone, sometimes you still hit limits. This is especially common when you want many hundreds of instances, or when you want very large instances, or when you want GPUs. All of these have limited spot availability. 

Sometimes this is fine. Sometimes it isn't. Sometimes you need things to run predictably with the requested number of workers at the requested time. Our solution to this is to use spot but fall back to using on-demand if spot isn't available. We offer three different policies

- Spot: If you're very cost conscious and never want to pay the full price for your compute, we'll only give you spot instances. 
- Spot-with-fallback: good if you're using spot instances but it's also important to you that you get clusters of a certain size and are willing to pay full price as necessary.
- On-demand: which we've pretty much stopped using, but which you could use if you want to be very very sure that everything is stable.

We've switched our internal Dask benchmarking to use spot instances with fallback. As I mentioned before, we get spot instances about 99.9% of the time—but for the remaining 0.1% we get on-demand fallback instances so that there's no impact on our benchmarks from the ups and downs of spot availability.

On-demand is currently the default, we're strongly considering making spot-with-fallback the default.

## 3 – Instance replacement

Are there any downsides to using spot with fallback compared to using only on-demand instances?

Unfortunately there are.

Spot instances aren't just harder to get, they're also harder to keep—the cloud provider can "interrupt" (AWS) or "preempt" (Google Cloud) your instances when they need to free up availability for other customers.

So how do we address this?

Coiled relies on Dask for scheduling work on your cluster. Dask's architecture is to have a single scheduler (we run this on its own on-demand instance) and potentially very many workers (this is where you get scale). The scheduler needs to stay up for the entire duration of your workload, but Dask is resilient to workers coming and going—this makes it easy to scale up and down in response to changes in your workload *or changes in cloud provider instance availability.*

We **always use on-demand for the scheduler** even when there's available spot capacity. On the one hand, that means you're paying full price for the scheduler instance. But on the other hand, the scheduler rarely needs to be a large instance—for our benchmarking on AWS we use an m6i.large—and there's only one scheduler, whereas there are usually many workers.

For **workers** we watch for advanced notification that a spot worker is going to be terminated, and we attempt to then gracefully shut it down so that computed results can be transferred to other workers. At the same time, we start spinning up another instance—which may be a spot instance, but if none are available and you've enabled fallback, it may be an on-demand instance. Dask is able to handle worker replacement midstream, so your workloads are able to keep running.

## Smooth User Experience for Cost Savings

Coiled makes it easy to quickly and reliably spin up clusters in the cloud to run your Python code in parallel at scale. Because we remove the hassle of manually provisioning and configuring servers and infrastructure, it's trivial for you to get a cluster with hundreds of cores and terabytes of memory that you might only need for a few minutes.

Before making the changes above, we found that while spot provided substantial cost savings in theory, it wasn't really usable. Users kept not getting the machines they needed, or their workloads would crash unpredictably. In practice folks stopped using spot and kept on with on-demand instances. Today folks are pretty happy. We use the default configuration in our own testing suites and our internal bills have gone down. Same with many of our customers.  

You can see [our docs](https://docs.coiled.io/user_guide/costs/spot.html) for the relevant keyword arguments when creating Coiled clusters.