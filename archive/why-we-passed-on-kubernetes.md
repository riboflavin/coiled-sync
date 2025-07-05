---
title: Why we passed on Kubernetes
description: Coiled deploys Dask clusters in the cloud. We do this using raw cloud APIs rather than with Kubernetes.
blogpost: true
date: 
author: 
---

# Why we passed on Kubernetes

Kubernetes is great if you need to organize many always-on services and have in-house expertise, but can add an extra burden and abstraction when deploying a single bursty service like Dask, especially in a user environment with quickly changing needs.

At Coiled we chose raw cloud APIs for flexibility and simplicity, and we encourage this choice for others in similar situations. This article explains this recommendation.

## Motivation

Coiled deploys Dask clusters in the cloud. We do this using raw cloud APIs rather than with Kubernetes. When new Dask users embark on resolving the deployment question they are often curious about using raw Cloud SaaS (like AWS EC2) vs Kubernetes. This article goes into some of the pros and cons behind this question.

For context, I've deployed Dask on many different Kubernetes systems. Kubernetes is fantastic and I often recommend it, but I've run into enough downstream pain that it's no longer my default recommendation.

## Deploying Dask on Kubernetes is Great

Kubernetes is great to deploy Dask for lots of different reasons:

1. **Great Dask deployment technologies exist including dask-kubernetes, dask-gateway, the helm chart, and QHub.**
2. **Kubernetes is the de-facto standard** deployment technology today. You're buying into a system that has strong buy-in and universal support from many different technologies.
3. **Kubernetes is cloud agnostic**, and so it's theoretically easy to shift your deployment to a different cloud vendor and avoid lock-in.
4. **Kubernetes is battle tested** and so you know that it'll just work, or that any problems will be your fault.
5. **Kubernetes allows you to share warm nodes** across services, allowing for faster uptime when you have a large userbase.
6. **Kubernetes can provide a single system for many services**, allowing you to amortize some of your organizational cost.

In general we find that when a large organization goes all-in on Kubernetes it can provide a lot of benefits.

## Dask Kubernetes is a Pain in the Butt

My experience is that setting up Dask clusters on Kubernetes initially is delightful, but that over time the Kubernetes abstraction layer provides more pain than benefit, at least when all you're trying to do is deploy Dask. Let's dig into why.

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/-VzaC5-Eh38?start=731&amp;enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Matthew Rocklin - Dask in Production | SciPy 2024" data-gtm-yt-inspected-34277050_38="true" id="356991061" data-gtm-yt-inspected-12="true"></iframe>

## Excess Abstraction

Most of the upcoming issues arise because Kubernetes tries to abstract away a system (the cloud) that you don't actually want to abstract away. Eventually you find yourself modeling much of the cloud VM / Networking stack in Kubernetes, which gets pretty painful.

## Instance Types

As an example, when you set up a Kubernetes cluster you'll back it with a node pool for which you will select a standard instance type, say, m5.4xlarge. This instance type determines:

1. The maximum size of a pod
2. The general ratio between CPU cores and memory
3. Local storage / disk technology used (local SSD or not)
4. Whether or not you have GPUs
5. Advanced networking that may exist

Early on, this will be fine. You don't really care about your instance type. Dask abstracts that away for you… kind of.

Invariably you will have a sharp user who wants to use [GPUs](https://docs.coiled.io/user_guide/gpu-job.html), or SSDs, or large-memory workers for cost savings, or spot instances, or [ARM instances](https://docs.coiled.io/blog/dask-graviton.html), or other aspects.

That's ok though! You can still use Kubernetes, you'll just set up a suite of different node pools, and use various Kubernetes labeling schemes to select between them based on the worker type requested. You can absolutely model out the complexity of cloud instance types in Kubernetes; it's just work.

## Scale Up / Scale Down

Dask workloads are bursty. Often you want to scale from zero to 100 machines as soon as possible, use them for twenty minutes, and then scale back down to zero. With cloud APIs this is easy. You ask for 100 instances, use them for a while, and then destroy them.

With Dask Kubernetes there is some interplay between Kubernetes and the underlying auto-scaling node pools. These systems are often defined by the underlying cloud infrastructure, and have various policies like "If there are excess nodes, wait a minute and then remove one". These policies make sense for the slowly varying services for which Kubernetes was designed, but can be catastrophically costly for bursty computational services like Dask.

Again, we can reconfigure this, but we find ourselves un-engineering a system that didn't need engineering in the first place.

## Always-on

Less relevant for larger users, but for smaller groups just running the Kubernetes cluster sometimes does have some non-trivial cost. EKS has about a baseline cost of $0.10 per hour, or roughly $1,000 per year (GKE has a similar base cost). This is often less than computational costs for any corporate user, but isn't great for individuals or smaller groups.

## Maintenance

Kubernetes is great, but it does need tending. Invariably we see data professionals start down the Kubernetes path, have good early success, and then within about six months be spending about half of their time tending and modifying the cluster. This is, frankly, how many of the Dask engineers who work on distributed deployment changed careers.

The human cost is by far the largest monetary cost that we see for leveraging Kubernetes for Dask deployment.

## Multi-Region

When dealing with larger organizations, or multiple organizations, one tends to need to manage multiple regions. When using raw cloud APIs it's pretty easy to say "give me 100 workers in us-west-1 or eu-west-2". When using Kubernetes you tend to create new Kubernetes clusters in each region, which compounds the issues above.

## Do I Need Kubernetes?

In general, we find that when organizations have bought into Kubernetes, have qualified professionals maintaining it, and run many always-on services on it then it makes sense to continue using it for bursty computational services like Dask. However when organizations are first starting, don't have lots of in-house experience, or can't amortize the maintenance cost across many similar services then Kubernetes is rarely the correct choice long-term.

A common challenge arises because Kubernetes is pretty easy to get started with, and so people start out, but then the complexities arise late and they're stuck with a sunken cost.

## Coiled's Decision

Given the experience above, it was obvious for Coiled, an always-on SaaS service provider that we would choose to host our computation with raw cloud APIs. Coiled provides a flexible service for a very wide variety of use cases. Our users want different regions, GPUs, [Spot](https://www.coiled.io/blog/save-money-with-spot), ARM chips, etc. and they want it all with rapid scale-up / scale-down properties without an ongoing cost.

Of course, making a highly robust service wasn't trivial, but now that this work is done it's pretty easy for the world to leverage it, a fact of which we are quite proud.