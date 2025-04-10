---
title: Tackling unmanaged memory with Dask
description: Unmanaged memory is RAM that the Dask scheduler is not directly aware of and which can cause workers to run out of memory and cause computations to hang and crash.
blogpost: true
date: 
author: 
---

# Tackling unmanaged memory with Dask

**_TL;DR:_ _unmanaged memory is RAM that the Dask scheduler is not directly aware of and which can cause workers to run out of memory and cause computations to hang and crash._**

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kG9SjBqg4MFS-65xkgajJUf-V0fOQ&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=9cec684d-d3c3-4332-b15a-e37b634c7771&amp;redirect_url=APefjpEa-uyEF9pPoFbOhhKDgZv0mVRHzqljRWICicl5DnY9CgjuIsZfIMHL9emiGXQU4A7BCELvS9fq61ibSi-kkxNLZ5c4alyVkeEJkOOh2eXW7q9j3-FwdPbbvN20Q4epFC05T0cC&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Ftackling-unmanaged-memory-with-dask&amp;ts=1744162586146" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>

In this article we:

- Shed light on the common error message "Memory use is high but worker has no data to store to disk. Perhaps some other process is leaking memory?"
- Identify the most common causes of unmanaged memory
- Use the Dask dashboard to monitor it
- Learn techniques to deflate it in some of the most typical cases
- Get an overview of what's changing in how Dask deals with unmanaged memory

## What is managed memory?

At any given moment, the Dask scheduler tracks chunks of data scattered over the workers of the cluster, which is typically either the input or output of Dask tasks. The RAM it occupies is called _managed memory_.

For example, if we write:

```python
from dask.distributed import Client
import numpy
client = Client()
future = client.submit(numpy.random.random, 1_000_000)
```

Now one of the workers is holding 8 MB of managed memory, which will be released as soon as the future will be dereferenced.

## What is unmanaged memory?

Managed memory never fully accounts for all of the _process memory_, that is the RAM occupied by the worker processes and observed by the OS.

The difference between process memory and managed memory is called _unmanaged memory_ and it is the sum of:

- The Python interpreter code, loaded modules, and global variables
- __sizeof__() method of objects in managed memory returning incorrect results
- Memory temporarily used by running tasks (sometimes improperly referred to as _heap_)
- Dereferenced Python objects that have not been garbage-collected yet
- Unused memory that the Python memory allocator did not return to libc through free() yet
- Unused memory that the user-space libc free() function did not release to the OS yet (read _Memory trimming_ below)
- Memory leaks (unreachable memory in C space and/or Python objects that are never released for some reason)
- Memory fragmentation (large contiguous objects - e.g. numpy arrays - failing to fit in the empty space left by deallocated smaller objects; large memory pages that are mostly free but can't be released because a small part of them is still in use)

In an ideal situation, all of the above are negligible in size. This is not always the case, however, and unmanaged memory can accumulate to unsustainable levels. This can cause workers to run out of memory and cause whole computations to hang or crash. You may have experienced this yourself. For example, when using Dask you may have seen error reports like the following:

Memory use is high but worker has no data to store to disk.  

Perhaps some other process is leaking memory?  

Process memory: 61.4GiB -- Worker memory limit: 64 GiB

## Monitor unmanaged memory with the Dask dashboard

Since distributed 2021.04.1, the Dask dashboard breaks down the memory usage of each worker and of the cluster total:

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b59687ae47a13a58bc2_image-9-700x789.png" alt=">

In the graph we can see:

- _Managed_ memory in solid color (blue or, if the process memory is close to the limit, orange)
- _Unmanaged_ memory in a lighter shade
- _Unmanaged recent_ memory in an even lighter shade (read below)
- Spilled memory (managed memory that has been moved to disk and no longer occupies RAM - see [https://distributed.dask.org/en/latest/worker.html#spill-data-to-disk](https://distributed.dask.org/en/latest/worker.html#spill-data-to-disk)) in grey

_Unmanaged recent_ is the portion of unmanaged memory that has appeared over the last 30 seconds. The idea is that, hopefully, most of it is due to garbage collection lag or tasks heap and should disappear soon. If it doesn't, it will transition into unmanaged memory. Unmanaged recent is tracked separately so that it can be ignored by heuristics - namely, rebalance() - for increased stability.

By construction, **process memory = managed + unmanaged + unmanaged recent**.

The same information is available in tabular format on the _Workers_ tab:

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5a040abd1604964aa0_image-15-700x502.png" alt=">

In the screenshots above, we see large amounts of _unmanaged recent_ memory - which should hopefully help us identify what event caused it.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kG9SjBqg4MFS-65xkgajJUf-V0fOQ&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=9cec684d-d3c3-4332-b15a-e37b634c7771&amp;redirect_url=APefjpEa-uyEF9pPoFbOhhKDgZv0mVRHzqljRWICicl5DnY9CgjuIsZfIMHL9emiGXQU4A7BCELvS9fq61ibSi-kkxNLZ5c4alyVkeEJkOOh2eXW7q9j3-FwdPbbvN20Q4epFC05T0cC&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Ftackling-unmanaged-memory-with-dask&amp;ts=1744162586146" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>

## Investigate and deflate unmanaged memory

If you see a large amount of unmanaged memory, it would be a mistake to blindly assume it is a leak. As listed earlier, there are several other frequent causes:

### Task heap

How much RAM is the execution of your tasks taking? Do large amounts of unmanaged memory appear only when running a task and disappear immediately afterwards? If so, you should consider breaking your data into smaller chunks.

### Garbage collection

Do you get a substantial reduction in memory usage if you manually trigger the garbage collector?

```python
import gc
client.run(gc.collect)  # collect garbage on all workers
```

In CPython (and unlike, namely, pypy and Java) the garbage collector is only involved in the case of circular references. If calling gc.collect() on your workers makes your unmanaged memory drop, you should investigate your data for circular references and/or tweak the gc settings on the workers through a Worker Plugin [[https://distributed.dask.org/en/latest/plugins.html#worker-plugins](https://distributed.dask.org/en/latest/plugins.html#worker-plugins)].

### Memory trimming

Another important cause of unmanaged memory on Linux and MacOSX, which is not widely known about, derives from the fact that the libc malloc()/free() manage a user-space memory pool, so free() won't necessarily release memory back to the OS. This is particularly likely to affect you if you deal with large amounts of small Python objects - basically, all non-NumPy data as well as small NumPy chunks (which underlie both dask.array and dask.dataframe). What constitutes a "small" NumPy chunk varies by OS, distribution, and possibly the amount of installed RAM; it's been empirically observed on chunks sized 1 MiB each or smaller.

To see if trimming helps, you can run (Linux workers only):

```python
import ctypes
def trim_memory() -> int:
    libc = ctypes.CDLL("libc.so.6")
    return libc.malloc_trim(0)
client.run(trim_memory)
```

Here's an example of how the dashboard can look like before and after running malloc_trim:

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5a040abd76b7964a9f_image1-3-700x407.png" alt=">

If you observe a reduction in unmanaged memory like in the above images, read [https://distributed.dask.org/en/latest/worker.html#memtrim](https://distributed.dask.org/en/latest/worker.html#memtrim) for how you can robustly tackle the problem in production.

## What's changing in how Dask handles unmanaged memory

Starting from distributed 2021.06.0, rebalance()[[https://distributed.dask.org/en/latest/api.html#distributed.Client.rebalance](https://distributed.dask.org/en/latest/api.html#distributed.Client.rebalance)] takes unmanaged memory into consideration, so if e.g. you have a memory leak on a single worker, that worker will hold less managed data than its peers after rebalancing. This is something to keep in mind if you are both a rebalance() user and are affected by trimming issues (as described in the previous section).

In the future, we will revise other heuristics that measure worker memory - namely, those underlying scatter() [[https://distributed.dask.org/en/latest/locality.html#data-scatter](https://distributed.dask.org/en/latest/locality.html#data-scatter)]

and replicate() [[https://distributed.dask.org/en/latest/api.html#distributed.Client.replicate](https://distributed.dask.org/en/latest/api.html#distributed.Client.replicate)] to ensure that they consider unmanaged memory.

## Conclusions

Recent developments in Dask made memory usage on the cluster a lot more transparent to the user. Unexplained memory usage is not necessarily caused by a memory leak, and hopefully, this article added some tools to your kit when you need to debug it.

To learn more about Dask, visit the Dask website by clicking below.

## Try Coiled for Free

Thanks for reading. If you're interested in trying out Coiled, which provides hosted Dask clusters, docker-less managed software, and one-click deployments, you can do so for free today when you click below.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kG9SjBqg4MFS-65xkgajJUf-V0fOQ&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=9cec684d-d3c3-4332-b15a-e37b634c7771&amp;redirect_url=APefjpEa-uyEF9pPoFbOhhKDgZv0mVRHzqljRWICicl5DnY9CgjuIsZfIMHL9emiGXQU4A7BCELvS9fq61ibSi-kkxNLZ5c4alyVkeEJkOOh2eXW7q9j3-FwdPbbvN20Q4epFC05T0cC&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Ftackling-unmanaged-memory-with-dask&amp;ts=1744162586146" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>