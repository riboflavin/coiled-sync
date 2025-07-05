---
title: Better Shuffling in Dask: a Proof-of-Concept
description: This post outlines the Coiled team's recent experimentation with a new approach to DataFrame shuffling in Dask.
blogpost: true
date: 
author: 
---

# Better Shuffling in Dask: a Proof-of-Concept

*Updated May 16th, 2023: With release 2023.2.1, dask.dataframe introduced this shuffling method called P2P, making sorts, merges, and joins faster and using constant memory. Benchmarks show impressive improvements. [See our blog post](https://blog.coiled.io/blog/shuffling-large-data-at-constant-memory.html).*

Over the last few weeks, the Coiled team has been experimenting with a new approach to DataFrame shuffling in Dask. It's not ready for release yet, but it does show a promising path forward for significantly improving performance, and we'd love it if you [tried it out](https://github.com/dask/dask/pull/8223)!

- Good news 👍 : our proof-of-concept can shuffle *much* larger datasets than were ever possible before. We've shuffled 7TiB DataFrames with 40,000 partitions and only needed a fraction of the full dataset size in total worker RAM.
- Bad news 👎 : the code is just a proof-of-concept, and not stable or maintainable enough to merge yet.

*This post goes into a lot of technical detail. If you just want to try out the new implementation, [skip to the end](#) for instructions.*

## Shuffling is easy

As discussed in [https://coiled.io/blog/spark-vs-dask-vs-ray](https://coiled.io/blog/spark-vs-dask-vs-ray), shuffling is a key part of most ETL workloads (namely the "transform" part, where the dataset is split up one way and needs to be reorganized along different lines). merge, set_index, groupby().apply(), and other common operations are all (sometimes) backed by shuffling.

In principle, shuffling is very simple: you look at a row in a DataFrame, figure out which output partition number it belongs to, figure out which worker should hold that partition in the end, and send the data along:

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5df9c63728c2ce7ac6_image-3-700x340.png" alt=">

*Illustration of how a distributed shuffle works in principle*

Oddly though, this operation that sounds like a simple for-loop is awkward to express as a [task graph](https://docs.dask.org/en/latest/graphs.html) like Dask uses. Every output partition depends on every input partition, so the graph becomes N² in size. Even with reasonable amounts of input data, this can crash the Dask scheduler.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5df9c6372f4bce7ac3_image7.png" alt=">

*The current task graph of a very small shuffle (20 partitions). It grows quadratically with the number of partitions, so imagine this times 100 or 1000—it gets large very quickly!*

And when it does work, the scheduler is still the bottleneck, trying to micromanage the transfer of these many sub-fragments of data. Since there's no heavy number-crunching involved, the limiting factor should be IO: sending data over the network, and to disk if shuffling a larger-than-memory dataset. However, Dask's current implementation isn't efficient enough to reach these hardware limits.

## A new approach to shuffling in Dask

So we tested a rather heretical idea to make Dask's shuffling better: just bypass a lot of Dask. We wrote an extension that attaches to every worker and uses the worker's networking APIs to send data directly to its peers without scheduler intervention. We still used a task graph, letting us lean on the centralized scheduler for some synchronization that's otherwise tricky to handle in a distributed system, but effectively hid the low-level details from the scheduler, keeping the graph small (O(N) instead of O(N²)).

We confirmed two wholly unsurprising things:

1. A dedicated, peer-to-peer, decentralized implementation is much more performant and gives us the low-level control to manage memory, network, and disk better than the centrally-scheduled equivalent can.
2. Bypassing the task graph in a system designed around task graphs leads to some pretty hacky code.

This is great to confirm! Because it tells us:

1. We're not condemned to bad shuffles forever—there's a path forward.
2. It may be worth it to adjust the system so that bypassing task graphs isn't a hack, but a supported (and tested and maintained) feature.

The second won't be a radical change, or even something any but the most advanced end-users would ever do. Rather, it's a practical recognition that Dask's goal is to make scalable parallel computing broadly accessible to Python users, and task graphs are not the only way to do that. Graphs serve Dask very well, but like every tool, they're not perfect for every situation. In select situations that are both very important and very poorly suited to task graphs, we can help users a lot by not clinging too tightly to existing abstractions.

## Actually, shuffling is hard

With that philosophizing out of the way, we'll talk about an assortment of interesting challenges we ran into with this new system.

Though the shuffling algorithm is simple in principle, doing it well takes some Goldilocks-like balancing of resources (memory, network, disk, concurrency). Both network and disk perform better in batches, rather than lots of tiny tiny sends/writes. So we added logic to accumulate pieces of the input partitions until we have enough to send to a given worker and to accumulate those received pieces until we have enough to write to disk. Buffering like this while not running out of memory is a balancing act. In our final implementation, we hope to make this both simpler and more self-tuning.

But it turns out the whole thing may have been a premature optimization since network and disk are still far from being the bottleneck.

### GIL Gotchas

We profiled the shuffle using [py-spy](https://github.com/benfred/py-spy), which gave a detailed view of what was happening from our Python code all the way down to the lowest-level C code within the CPython standard library. This quickly showed we were spending 70% of our time waiting on the GIL (Global Interpreter Lock)!

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5df9c6373a54ce7ac7_image-4-700x488.png" alt=">

*py-spy profile of our new shuffle running on a worker. The beige pthread_cond_timedwait at the bottom of the stacks is time spent blocked waiting for the GIL.*

Since we're writing a multi-threaded program, this is concerning in general—it means we won't be able to get much parallelism out of it. Our worker threads (which split and re-group DataFrames as they're sent or received) call pd.concat a lot, which internally calls np.concatenate, which, it turns out, doesn't release the GIL. So this is [one problem](https://github.com/pandas-dev/pandas/issues/43155#issuecomment-915689887): we can't parallelize our CPU work as we'd hoped to.

But the bigger problem is that this is slowing down our networking enormously. We're using non-blocking sockets, so the calls to send and receive are supposed to be nearly instantaneous and—as the name suggests—non-blocking. But because these calls actually have to block on re-acquiring the GIL, they might take 10x longer than they need to, gumming up the asyncio event loop (the cardinal sin of asynchronous programming). In the end, this prevents the *next* send or receive from running as soon as it should have, and so on, such that we complete these sends and receives far less frequently than we want to.

This is a well-known problem in Python that's been debated since 2010 ([bpo-7946](https://bugs.python.org/issue7946)) known as the "convoy effect". For an excellent in-depth explanation of this, see [this blog post](https://tenthousandmeters.com/blog/python-behind-the-scenes-13-the-gil-and-its-effects-on-python-multithreading/). But the premise is something like this:

There are hundreds of people in a house who are sharing one bathroom. Most of them want to shower (slow). Some of them just want to brush their teeth (fast). But the tooth-brushers are very considerate, and recognize that while they're brushing their teeth, they don't actually need to be in the bathroom—they could step outside, then come back in when they're done to spit in the sink. But what happens is that when they step out, usually a showerer immediately jumps in and hogs the bathroom for a long time while singing show tunes, leaving the tooth-brusher awkwardly waiting around through a rendition of the entire soundtrack to *Hamilton*. Overall, it means the tooth-brushing takes far longer than it needs to, and if the tooth-brushers had just been a little less considerate, they could have finished much faster and nobody would have complained much.

As much as this sounds bleak (a known issue for over a decade! in CPython! labeled as wontfix!), I'm actually very excited and optimistic about this. We've understood the root problem, and it means we can stop worrying about optimizing other stuff for now, since it won't have much of an effect. And I think there are opportunities for both an "avoid it" solution (work with NumPy devs to release the GIL during concat? avoid pandas and NumPy altogether, and use Arrow/polars/etc. instead?) and a "fix it" solution (don't release the GIL with non-blocking sockets, either in CPython or in an alternative library like uvloop?). And either approach will benefit not just shuffling, but Dask as a whole, and likely the entire Python community.

*As a side note, this particularly shows how valuable good profiling tools like py-spy are. If we'd just looked at the Python call stack (say, through Dask's built-in profiler), we might have seen read and write took most of the time and concluded that we were network bound, and the network was just slow. But by being able to look deeper, we discovered that most of that supposed networking time was actually spent doing nothing at all.*

### Pandas concatenation

Besides the GIL issue, we found concatenating many pandas DataFrames was slower than we expected. In [conversations with pandas developers](https://github.com/pandas-dev/pandas/issues/43155), they've identified a couple of issues that will hopefully make their way into performance improvements in future pandas versions.

### Root task overproduction

Another interesting problem was that Dask workers would run out of memory because they loaded more input DataFrames upfront than they needed to, instead of loading and immediately starting to transfer them to other workers. This turns out to be a rather core consequence of the way distributed scheduling is designed in Dask, and is an issue that's been bothering many Dask users [for a while](https://github.com/dask/distributed/issues/2602), and has been [proposed to fix](https://github.com/dask/distributed/issues/3974) (but the fix is a complex change).

You can read about the problem in more detail [in this issue](https://github.com/dask/distributed/issues/5223), but the premise is that the scheduler tells workers about all of the root tasks upfront, but doesn't tell them about downstream tasks until it hears that the corresponding root tasks have completed. In the time gap between a root task completing and the scheduler telling the worker, there's something else to use that root data for, the worker goes off and runs extra root tasks since that's all it knows about.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5df9c6373120ce7ac5_image-6-700x590.png" alt=">

*How root tasks get over-produced*

This is an important issue, and we're excited to solve it. However, for this particular case, we realized there was a simple workaround: the "fusion" step of graph optimization. By fusing the "load the data" root task to its single "transfer the data" dependent task (producing one "load and transfer the data" root task), it didn't matter if workers ran too many of those tasks initially. Thanks to [recent work](https://blog.dask.org/2021/07/07/high-level-graphs) on High-Level Graphs, with a slight tweak to the graph, this fusion step happens automatically and will fuse across most DataFrame operations.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5df9c63735ffce7ac4_image-7-700x320.png" alt=">

*Task fusion to the rescue!*

We're glad this workaround was easy, since we expect that delivering effective shuffling can be done a bit faster than solving this underlying issue, and it will help lots of users. But addressing this root task overproduction issue is very high on the priority list after that.

## Trying this out

If you've been struggling with large set_index or groupby().apply() operations failing, we'd love to get some early feedback on this proof-of-concept PR. Keep in mind that this is experimental and has some very sharp edges; see the warnings listed at [https://github.com/dask/dask/pull/8223](https://github.com/dask/dask/pull/8223). Nonetheless, it does work on straightforward cases, so if it works for you—or it doesn't—we'd love to hear the feedback as comments on that PR (or new issues, if you encounter specific problems).

Here's how you can try out the new shuffle implementation on Coiled.

First, create a local virtual environment and install the versions from git:

```bash
$ conda create -n shuffle -c conda-forge dask coiled jupyterlab dask-labextension
$ conda activate shuffle
$ pip install -U --no-deps git+https://github.com/gjoseph92/dask@shuffle_service git+https://github.com/dask/distributed
```

Then spin up a cluster, letting [package sync](https://docs.coiled.io/user_guide/package_sync.html) handle replicating your environment in your cluster:

```python
import dask
import coiled

cluster = coiled.Cluster(n_workers=10, worker_cpu=2, worker_memory="4GiB")
client = cluster.get_client()
client.wait_for_workers(10)
```

Now make some example data, and try re-indexing it.

Be sure to open the cluster dashboard at [https://cloud.coiled.io](https://cloud.coiled.io) to see what's going on.

```python
df = dask.datasets.timeseries("2000-01-01", "2005-01-01")
dfz = df.assign(z=df.x + df.y)
dfz.set_index("z", shuffle="service").size.compute()
```

Try removing the shuffle="service" to see how much of an improvement this makes!

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5df9c6378706ce7ac8_image2-700x398.png" alt=">

*Before: re-indexing using the current task-based shuffle.*

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5df9c637aa0bce7ac9_image1-700x398.png" alt=">

*After: using the experimental new shuffle*.

Here's a before-and-after of the current standard shuffle versus this new shuffle implementation. The most obvious difference is memory: workers are running out of memory with the old shuffle, but barely using any with the new. You can also see there are almost 10x fewer tasks with the new shuffle, which greatly relieves pressure on the scheduler.

Thanks for reading! And if you're interested in trying out Coiled Cloud, which provides hosted Dask clusters, docker-less managed software, and one-click deployments, you can do so for free today when you click below.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="