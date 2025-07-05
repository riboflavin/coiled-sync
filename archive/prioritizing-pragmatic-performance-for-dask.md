---
title: Prioritizing Pragmatic Performance for Dask
description: Dask developers care about performance, we've always taken a pragmatic rather than exciting approach to the problem...
blogpost: true
date: 
author: 
---

# Prioritizing Pragmatic Performance for Dask

Many people say the following to me:

*You should focus on X in Dask!  X makes things really fast!*

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/637473192b9adb450419d4ac_Logo%20Full%20Color.svg" loading="lazy" alt="Dask logo">

Some examples of X:

- Polars
- High level query optimization
- C/C++/Rust
- POSIX shared memory
- Reducing memory copies
- Massive scale

In some cases I think that they're absolutely right. In other cases I agree that the thing that they're talking about is fast, but I don't think that it will meaningfully improve common workloads, especially in a distributed cloud computing environment.

Instead, I tend to orient development pretty strongly around what profiling, benchmarking, and user engagement say is the biggest bottleneck, and go from there. This points me to topics like the following:

- Improving S3 bandwidth
- Intelligent scheduling to avoid memory buildup
- Removing Python object dtypes
- Removing Python for loops from user code
- Better observability to help users know what's going on

This means that Dask often loses in flat-out performance benchmarks in artificial scenarios, but that it continues to be the standard of use when it comes to scalable Python data processing.

This article will go into a few things that I think are great ideas, but are not yet major bottlenecks and why. Then it will go into a few things that I think are less-than-exciting-but-really-important improvements for common workloads.

## Things people ask for

#### Anything that is faster than S3

First, many workloads are bound by S3/GCS/object-store read performance. This tends to be 200 MB/s per node (with variation). If the thing that you're talking about is much faster than this then it probably doesn't matter that much (unless you're doing it hundreds of times per byte (in which case you're maybe doing something wrong)).

#### Memory Copies

*Memory copies make things slow, but only by a tiny bit*

Memory bandwidth often operates at 20 GB/s on a modest machine. So while it's true that memory copies slow things down, they only slow them down a little bit. It's like asking a racecar to go once more around the racetrack when we're still waiting for a donkey (s3/disk/network) to finish going around once.  It just doesn't matter.

But for the sake of argument let's imagine for a moment that you're at a performance point where this *does* matter. That means that you are processing data at 20 GB/s on a modest machine, or around 1 TB per minute. Per day you're processing about 1 PB on this single node.

If you're in that world, then you either …

1. Generate many petabytes of data per day
2. Don't need parallel computing, and so don't need Dask (Hooray!  Fewer tools!)

#### High Level Query Optimization

*Doing less work is good*

In contrast, I think that high level optimization is very very good. For one, it often means that you get to reduce the amount of data that you load, which reduces the demand on the slow parts of your system (like S3). Dask Dataframes/Arrays doesn't do this well today, but it should.

*Updated August 2023: You can now try out Dask Expressions with pip install dask-expr. Learn more in [our blog post](https://medium.com/coiled-hq/high-level-query-optimization-in-dask-995640564ed7).*

#### C/C++/Rust

*You're already using these for 99% of your computation, the last 1% doesn't matter much*

If you're doing things well then you're already relying on mostly low level code (numpy, pandas, pytorch, scikit-learn, arrow, …) The code calling those routines accounts for a small fraction of your total runtime and is not worth optimizing.

Said another way, most data workloads are saturated by network and disk. I can, with Python, saturate network and disk and so go as fast as possible. If we're in this regime then, unless a faster language can somehow make your disk/network hardware go faster, then language choice doesn't matter.

#### Polars

Taking a look at the options above, I'm really excited about Polars mostly due to the high level query optimization. The other bits are nice, but mostly on single machines where you have really fast access to disk and don't need to worry about network as much. 

Ritchie Vink (polars core dev) and I actually started a [dask-polars project](https://github.com/pola-rs/dask-polars) several months ago. I'd love to work more on it (it's fun!).  I'm waiting for this need to come up more in benchmarks before I personally prioritize it more.

The best arguments that I've heard for prioritizing it early so far are …

1. High level query optimization (Dask dataframes should have this, and don't)
2. Some folks like the API more than Pandas

The more I hear folks report #2 above the more I'll shift efforts over to it. So far I'm seeing 50x more Pandas usage (but down from 1000x a few months ago, so the trend here is worth watching).

To be clear though, folks should use Polars. It helps to avoid the distributed computing problem significantly. Polars is great.

#### POSIX Shared Memory

This mostly comes from ray users who like the idea of sharing memory between processes. The conversation usually goes like this:

- Them: Dask should use POSIX shared memory so that you can share data between cores!
- Me: Dask does that today!  We use a single normal process with lots of normal threads and normal shared memory!
- Them: But then what about the GIL?!
- Me: The GIL isn't an issue!  The GIL has been released in most PyData libraries ever since Dask came out seven years ago!  Here is an [ancient blogpost](https://matthewrocklin.com/blog/work/2015/03/10/PyData-GIL) on the topic!

Threads + normal memory is less fancy, but also way simpler than posix-shared-memory!

- Them: Hrm, maybe this is important if people are writing lots of for-loopy python code?
- Me: That's true!  But that's also a 100x performance hit! I encourage you to address that much much deeper issue before trying to parallelize it. Maybe consider teaching the pandas API or using Numba?

Now, this is really important if you're doing distributed training of very large machine learning models. Almost no one I talk to is doing this though. Mostly this issue comes up because, I think, people misunderstand the GIL.

To address this misunderstanding, I think that we need to improve observability around the GIL. See [https://github.com/dask/distributed/issues/7290](https://github.com/dask/distributed/issues/7290).

But let's say that folks here are right and that we do want to reduce intra-machine communication costs with shared memory. How much of a savings is this, actually? The common alternative is just to ask for many more smaller machines. It's true that inter-machine communication is more expensive than intra-machine communication, but most communication is going to be inter-machine anyway, so the savings here is minimal. There's just no way that this respects [Amdahl's law.](https://en.wikipedia.org/wiki/Amdahl%27s_law)

#### Massive Scale

*We work at really big scale. What if we want to scale to thousands of machines?*

Well, Dask actually works fine at thousands of machines. You'll probably want to change some of the default configuration; let's chat. Really though, if you're doing things well then you probably don't need thousands of machines.

I work with some of the largest organizations doing some of the largest computations. It is *very rare* for someone to need thousands of machines. It is much more common for them to think that they need thousands of machines, get feedback from Dask about how their workflow is running, and then learn that they can accomplish more than they wanted with far fewer resources.

Most Dask clusters in large organizations operate at the 10-500 machines range with the bottom of that range being far more common. Some folks get up to a thousand machines. Even at this point the scheduler can be idling most of the time if you're doing things efficiently.

## Things I'm excited about

Most dataframe workloads that are slow are due to one of these things:

1. Running out of memory
2. Object dtypes
3. Iterating over rows with a for loop
4. Network/disk bandwidth 
5. Loading in data we didn't need 

This covers about 95% of slow workflows. People who get rid of these issues are invariably happy. Let's talk through a few things that are in flight today that can be done to address them.

#### Task Queuing / more intelligent memory scheduling 

The Dask scheduler is *very* smart about placing and prioritizing tasks to reduce overall memory use. If you are running into a workload that you think should run in small memory but doesn't then I encourage you to try out the 2022.11.0 release. It's pretty awesome. We've written more about it [in this blog post](/blog/reducing-dask-memory-usage).

Here is a snippet from the last Dask Demo Day with engineer Florian Jetter talking about it and showing the impact:

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/VlTgcLqb1DQ?start=516&amp;enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Dask Demo Day - 2022-10-27" data-gtm-yt-inspected-34277050_38="true" id="179025058" data-gtm-yt-inspected-12="true"></iframe>

#### PyArrow Strings

*Updated August 2023: as of **Dask 2023.7.1, PyArrow backed strings are the default in Dask DataFrames.** This option is automatically enabled if you have at least pandas 2.0 and PyArrow 12.0 installed. See the [Dask release notes](https://docs.dask.org/en/stable/changelog.html#v2023-7-1) for implementation details and our [blog post](https://www.coiled.io/blog/pyarrow-strings-in-dask-dataframes) on how this change improves performance.*

Python is slow. Normally we get past this by writing performant code in C/C++/Cython and then linking to it. Pandas does this, but unfortunately they didn't have a good way to handle strings, so they stayed with Python for that kind of data. 😞

Fortunately, Arrow handles this. They have nice strings implemented in C++. This results in lower memory use and faster speeds. Here is a demonstrative video:

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/_zoPmQ6J1aE?start=151&amp;enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Pandas/Arrow/Dask String Performance Improvements | Matt Rocklin" data-gtm-yt-inspected-34277050_38="true" id="396660511" data-gtm-yt-inspected-12="true"></iframe>

For a long time PyArrow strings were exciting, but not fully supported. It seems like today they are. Pandas itself is considering switching over by default. My guess is that Dask may lead pandas here a bit and make the switch earlier. This is in active development.

#### Extension data types more generally

This also holds for other data types, including nullable dtypes (ints, bools, etc..), decimals, and others. There's good movement towards just switching over generally. 

Pandas 2.0 is coming out soon (rc branch already tagged) and in general I expect to see a big shift in this direction over the next year.

#### S3 Bandwidth

Sometimes we don't handle things as well as we should. Recently we found a nice 2x speed boost. 

[https://github.com/dask/dask/issues/9619#issuecomment-1302306306](https://github.com/dask/dask/issues/9619#issuecomment-1302306306)

#### Shuffle service

When you want to sort/set-index/join dataframes you often need to do an all-to-all shuffle. Dask's task scheduling system wasn't designed for this and while it does it, it does it poorly in a way that often stresses the scheduler and bloats memory.  

There is an experimental shuffle service (similar to what Spark uses) in Dask today that has solid performance, but doesn't yet have rock-solid-reliability (and so is not yet default). We hope to have this in by early 2023.  

#### High Level Query Optimization

*Updated August 2023: You can now try out Dask Expressions with pip install dask-expr. Learn more in [our blog post](https://medium.com/coiled-hq/high-level-query-optimization-in-dask-995640564ed7).*

Probably sometime in 2023 we should focus on updating our high level graph specification, which grew organically and has a bit of cruft. After this we should be able to start development on basic query optimizations. This plus shuffling and good dtype support should get us into a point where Dask dataframes are genuinely competitive performance-wise in ways that matter.

Additionally, high level query optimization should also be applicable to other Dask collections, like Arrays/Xarray, which should be fun.

#### Things that aren't performance

It's worth pointing out that most distributed data computing users I meet would happily trade in a 3x performance hit if things would "just work". Topics like ease of deployment, visibility, stability, and ease-of-use for less technical teammates often supersede a desire for things to go fast.

Dask has never been the fastest framework (go to MPI for that) but we have probably always been the easiest to use and understand and adapt to user needs. This results in a joint human+computer performance that can't be beat.

## Summary

Dask developers care about performance, we've always taken a pragmatic rather than exciting approach to the problem.  We interact with users, identify user pain, and focus on that pain. 

This focus on user pain has served us well in the past, but often orients us differently from the focus of the public, which is understandably excited by shiny developments. I encourage folks to profile and benchmark their workloads (Dask provides excellent tools for this) and [let us know what you see](https://github.com/dask/dask/issues/new). We're excited for more input.