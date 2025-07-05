---
title: Easily Run Python Functions in Parallel
description: When you search for how to run a Python function in parallel, one of the first things that comes up is the multiprocessing module. The documentation describes parallelism in terms of processes versus threads and mentions it can side-step the infamous Python GIL (Global Interpreter Lock).
blogpost: true
date: 
author: 
---

# Easily Run Python Functions in Parallel

When you search for how to run a Python function in parallel, one of the first things that comes up is the multiprocessing module. The [documentation](https://docs.python.org/3/library/multiprocessing.html) describes parallelism in terms of processes versus threads and mentions it can side-step the infamous Python GIL (Global Interpreter Lock).

This is all great if you're a Python developer comfortable navigating the intricacies of processes and threads. But what if you don't want to care about any of that? What if all you want is a straightforward way to run a Python function in parallel?

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/6787fa6db50494a9840ccc52_6787f7851641e2523d879e6f_processes-vs-threads.png" loading="lazy" alt=">

Different models of concurrency, adapted from [https://realpython.com/python-concurrency](https://realpython.com/python-concurrency)

Imagine you're estimating the global burden of malaria. You don't want to dive into the technicalities of parallelism—you just want to answer a question like, "How many people died of malaria in 1984?"

That's where **Dask** comes in.

## Why Choose Dask Over Multiprocessing?

Dask is designed to make parallel computing in Python as seamless as possible. With Dask, you can:

- Run Python functions in parallel with **minimal code changes**.
- Easily scale from a single machine to a distributed cluster
- Focus on your **data and analysis** instead of debugging processes and threads.

Whether you're analyzing global health data, processing large datasets, or running computational models, Dask lets you spend more time on your problem and less time wrestling with low-level parallelism.

## Run a Python Function in Parallel

You can use Dask to run any Python function in parallel. Let's say you have a function called `costly_simulation` that defines a long-running simulation over a set of parameters. With Dask, we can run this in parallel locally on all cores available on your laptop:

```python
from dask.distributed import LocalCluster

cluster = LocalCluster()                                # Use all cores on local machine
client = cluster.get_client()

parameters = [...]
def costly_simulation(parameter):
    ...

futures = client.map(costly_simulation, parameters)     # Run simulation in parallel
results = client.gather(futures)
```

Or scale out to a cluster of many machines on the cloud using Coiled:

```python
import coiled

cluster = coiled.Cluster(n_workers=50)                  # Start 50 machines on AWS        
client = cluster.get_client()

parameters = [...]
def costly_simulation(parameter):
    ...

futures = client.map(costly_simulation, parameters)     # Run simulation in parallel
results = client.gather(futures)
```

Coiled runs from your AWS, Google Cloud, or Azure account and you'll get 10,000 free CPU-hours each month. It's free to [sign up](https://docs.coiled.io/user_guide/setup/index.html).

## Parallel For Loop with Dask

Here's an illustrative example doing a nested for loop over parameters and data files. In this example, we work sequentially, loading the data in the outer loop and scoring in the inner loop for all pairs.

```python
def score(params: dict, data: object) -> float:
    ...

results = []
for filename in filenames:             # Nested for loop
    data = load(filename)              # Load data in outer loop
    for param in params:               # Score in inner loop over all pairs
        result = score(param, data)  
        results.append(result)

best = max(results)                    # Get best score
```

‍

Working sequentially can be quite slow, especially if you're working through hundreds of files. We can use Dask to run this in parallel, using all cores available on your laptop:

```python
from dask.distributed import LocalCluster

cluster = LocalCluster()                 # Use all cores on local machine
client = cluster.get_client()

futures = []
for filename in filenames:               # Nested for loop
    data = client.submit(load, filename)  # Load data in outer loop
    for param in params:                 # Score in inner loop over all pairs
        future = client.submit(score, param, data)      
        futures.append(future)

results = client.gather(futures)

best = max(results)                      # Get best score
```

‍

If your data is stored on the cloud or you need access to hardware you don't have available locally (like more memory or GPUs) you can use Coiled to run the same workflow across a cluster of VMs:

```python
import coiled

cluster = coiled.Cluster(n_workers=100)   # Scale out to 100 machines
client = cluster.get_client()

futures = []
for filename in s3_filenames:            # Nested for loop
    data = client.submit(load, filename)  # Load data in outer loop
    for param in params:                 # Score in inner loop over all pairs
        future = client.submit(score, param, data)      
        futures.append(future)

results = client.gather(futures)

best = max(results)                      # Get best score
```

Coiled handles things like:

- [Python environment management](https://docs.coiled.io/user_guide/software/index.html)
- [Controlling cloud costs](https://docs.coiled.io/user_guide/costs/index.html)
- Combining hardware and Python-specific metrics into a single UI

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/554gEk_qHFk?enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Dask Futures for General Parallelism" data-gtm-yt-inspected-34277050_38="true" id="94519613" data-gtm-yt-inspected-12="true"></iframe>

‍

## When to Use Multiprocessing Instead of Dask?

While Dask is powerful and flexible, there are cases where multiprocessing might be a better choice:

- Standard Library: As part of Python's standard library, multiprocessing requires no additional installation or setup.
- Low Overhead: For smaller tasks, the overhead of Dask's scheduler might outweigh its benefits, making multiprocessing a more efficient option. Dask also has a [multiprocessing scheduler](https://docs.dask.org/en/stable/scheduling.html#local-processes) that is very lightweight for these situations.
- Fine-Grained Control: If you need detailed control over processes, shared memory, or inter-process communication, multiprocessing provides the tools you need. Though it's worth mentioning Dask is also flexible enough for this level of fine-grained tuning with tools like asynchronous futures, distributed locks, queues, etc. 

## Examples

If you're tired of fighting with multiprocessing or just want an easier way to run Python functions in parallel, Dask could be a good option. It's powerful, intuitive, and designed to help you focus on your work—not the underlying mechanics of parallel computing. For more examples using Dask for parallel Python you might consider the following:

- Process [1 TiB of arXiv S3 data](https://docs.coiled.io/user_guide/arxiv-matplotlib.html)
- Get a *really* good [estimation of Pi](https://docs.coiled.io/user_guide/pi.html)
- Train an [XGBoost model in parallel](https://docs.coiled.io/user_guide/xgboost.html)