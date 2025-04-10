---
title: Introducing the Dask Active Memory Manager
description: Dask release 2021.10.0 introduces the first piece of a new modular system called Active Memory Manager, which aims to alleviate memory issues.
blogpost: true
date: 
author: 
---

# Introducing the Dask Active Memory Manager

Historically, out-of-memory errors and excessive memory requirements have frequently been a pain point for Dask users. Two of the main causes of memory-related headaches are data duplication and imbalance between workers.

Dask release 2021.10.0 introduces the first piece of a new modular system called *Active Memory Manager*, which aims to alleviate memory issues. The examples in this article use Dask release 2021.11.2, which features improved API and tools.

## Data duplication

Whenever a Dask task returns data, it is stored on the worker that executed the task for as long as it's referenced by a *Future* or another task:

```python
from distributed import Client
client = Client(n_workers=2)
w1, w2 = client.has_what()
x = client.submit(lambda i: i + 1, 1, workers=[w1], key="x")
x.result() # Output: 2
client.has_what() # Output: {'tcp://127.0.0.1:37465': ('x',), 'tcp://127.0.0.1:44715': ()}
```

When a task runs on a worker and requires as its input data that was returned by another task on a different worker, Dask will transparently transfer the data between workers, ending up with multiple copies of the same data on different workers. This is generally desirable, as it avoids re-transferring the data if it's required again later on. However, it also causes increased overall memory usage across the cluster:

```python
y = client.submit(lambda i: i + 1, x, workers=[w2], key="y")
y.result() # Output: 3
client.has_what() # Output: {'tcp://127.0.0.1:37465': ('x',), 'tcp://127.0.0.1:44715': ('x', 'y')}
```

## Memory imbalances

Dask assigns tasks to workers following criteria of CPU occupancy, locality, and worker resources. This design aims to optimize the overall computation time; as a side effect, however, it can lead to some workers ending up with substantially higher memory usage than others.

## The Active Memory Manager

Active Memory Manager, or *AMM* for short, is a background routine that periodically runs in the scheduler and that copies, moves, and removes data on the Dask cluster. It is a plugin-based design where the core component listens to and then enacts *suggestions* from a list of policies. In other words, the policies implement arbitrarily sophisticated decisions regarding where data should be, while the AMM core component makes sure that these decisions are put in practice without compromising data integrity.

At the moment of writing, the Active Memory Manager is shipped with a single built-in policy, **ReduceReplicas**, which cleans duplicated data on the workers when it's no longer necessary. 

When it comes to choosing which copies to preserve or create and which to delete, the AMM always prefers preserving/creating on the workers with the lowest memory usage and deleting on those with the highest - thus rebalancing the cluster as it runs.

## Enabling the AMM

Since it is an experimental feature, the Active Memory Manager is disabled by default.

The simplest way to turn it on is through the dask config:

```python
distributed:
  scheduler:
   active-memory-manager:
    start: true
```

You can do the same with an environment variable before you start the scheduler:

```python
$ export DASK_DISTRIBUTED__SCHEDULER__ACTIVE_MEMORY_MANAGER__START=True
$ dask-scheduler
```

Or if you're using [Coiled](https://coiled.io):

```python
cluster = coiled.Cluster(
    environ={
        "DASK_DISTRIBUTED__SCHEDULER__ACTIVE_MEMORY_MANAGER__START": "True"
    }
)
```

You can also enable/disable/test it on the fly from the dask client:

```python
client.amm.start()
client.amm.running() # Output: True
client.amm.stop()
client.amm.running() # Output: False
```

With any of the above methods, the AMM will run all default policies (at the moment, just ReduceReplicas) every 2 seconds. If you're a power user, you can further alter the Dask configuration to change the run interval, cherry-pick and configure individual policies, or write your own custom policies.

You can read the full documentation at [https://distributed.dask.org/en/latest/active_memory_manager.html](https://distributed.dask.org/en/latest/active_memory_manager.html).

## A demonstration with dot product

The dot product between two matrices is a O(n²) problem where each point of the matrices needs to be paired with each other:

```python
import dask.array as da
A = da.random.random((40_000, 40_000), chunks=(4_000, 4_000))
b = (A @ A.T).sum()
```

A is a square matrix worth 12 GiB in total, split across 122 MiB chunks:

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5e3269c1bf809fe758_image-700x285.png" alt=">

If you run b.compute() on either the local threaded scheduler or on a single distributed threaded worker, the peak memory usage will be roughly 19 GiB; this is because no copies of the data need to be moved across workers. If you run the same problem on a cluster with multiple workers, however, you will necessarily incur data duplication. While the individual workers can mount a lot less than in the local use case, you should expect the cluster-wide occupation (the sum of the occupation on all the workers) to be substantially higher.

Let's use a free trial account on Coiled to quickly set up a cluster with 48 workers and 96 CPUs:

```python
import coiled
from distributed import Client

cluster = coiled.Cluster(
    n_workers=48,
    environ={
        "DASK_DISTRIBUTED__SCHEDULER__ACTIVE_MEMORY_MANAGER__START": "True"
    },
)
client = Client(cluster)
```

Let's test that the Active Memory Manager is running:

```python
client.amm.running() # Output: True
```

We're going to use [MemorySampler](https://distributed.dask.org/en/latest/diagnosing-performance.html#analysing-memory-usage-over-time) (new in Dask 2021.11.2) to record memory usage on the cluster:

```python
from distributed.diagnostics import MemorySampler
ms = MemorySampler()
```

Then we're going to run our computation on the cluster and fetch the history of our memory usage:

```python
with ms.sample("AMM on"):
    b.compute()
```

Now let's do it again, but this time without the Active Memory Manager:

```python
client.amm.stop()
with ms.sample("AMM off"):
    b.compute()
```

Finally, let's plot the data to see the difference:

```python
ms.plot(align=True, grid=True)
```

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5e3269c133bc9fe757_image-1-700x556.png" alt=">

```python
ms.to_pandas().max(axis=0) / 2**30

#
# Output:
#
# With AMM     135.177975
# No AMM       165.925610
# dtype: float64
```

The Active Memory Manager reduced the peak cluster-wide memory usage from 166 GiB to 135 GiB, with no runtime degradation!

## Next steps

ReduceReplicas is just the first of a series of AMM policies that will be released in the future:

- **Worker retirement** is being reimplemented to run on top of the AMM. The key benefit to this is that it will become possible to gracefully retire a worker on a busy cluster, while jobs are running on it. When a worker runs out of memory, it is automatically retired and restarted; this change will prevent random crashes in the jobs when that happens.
- Graceful worker retirement will also let you **partially downscale an adaptive cluster** before it's completely idle; this will potentially result in substantial monetary savings whenever a job features an initial burst of parallelism followed by a long "tail" of serial tasks, or when small jobs are continuously pushed to the cluster and there's a spike in usage (more jobs, or a much larger job) at a certain time of the day.
- **Worker pause** (which, by default, is triggered when a worker reaches 80% memory usage) is being redesigned to gracefully transition into retirement after a timeout expires.
- **Rebalance** is being reimplemented as an AMM policy and will automatically run every two seconds, while computations are running and without any need for user intervention. The manual method [Client.rebalance](https://distributed.dask.org/en/latest/api.html?highlight=rebalance#distributed.Client.rebalance) is going to be phased out.
- **Replicate** is also becoming an AMM policy. Today, [Client.replicate](https://distributed.dask.org/en/latest/api.html?highlight=replicate#distributed.Client.replicate) synchronously creates replicas, but does not track them after it returns; if a replica is lost later on, nothing will regenerate it unless the user manually invokes replicate() again. In the future, Client.replicate will inform the AMM of the desired number of replicas and immediately return; the AMM will start tracking them and ensure that, whenever a replica is lost, it is recreated somewhere else (as long as at least one copy of the data survives). 

You'll also be able to ask for a key to be replicated on all workers, including those that will join the cluster in the future.

## Thanks for reading!

If you'd like to scale your Dask work to the cloud, check out Coiled — Coiled provides quick and on-demand Dask clusters along with tools to manage environments, teams, and costs. Click below to learn more!

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kGjtBypQM47qRpYL0Hed5wAa4Rdhw&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=5dbe5a25-ae4f-4c17-a781-e2190bebd36c&amp;redirect_url=APefjpEnib68t4q_o9UQD4nrcEog4DQZ-BIRT3N3aOYE_fwToOa5m7dUSTXKfCuI4oSkSJXPkr58VOfj011LZ4gwSu9ZCeaIDfD4RDBaqQcesPHThPpHocpNgnR1LH8VRcohFooWtQkv&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fintroducing-the-dask-active-memory-manager&amp;ts=1744256063601" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>