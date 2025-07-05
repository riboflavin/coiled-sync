---
title: Dask for Parallel Python
description: 

Dask is a general purpose library for parallel computing. Dask can be used on its own to parallelize Python code, or with integrations to other popular libraries to scale out common workflows.
blogpost: true
date: 
author: 
---

# Dask for Parallel Python

Dask is a general purpose library for parallel computing. Dask can be used on its own to parallelize Python code, or with integrations to other popular libraries to scale out common workflows.

Dask has its own [docs](https://dask.org/), but we'll include a few typical use cases below.

## Big pandas

Dask DataFrames use pandas under the hood, so your current code likely just works. It's [faster than Spark](https://docs.coiled.io/blog/spark-vs-dask.html) and easier too. Here's an illustrative example of how you would load data from Parquet and perform some standard data manipulation with pandas:

```python
import pandas as pd

df = pd.read_parquet('s3://mybucket/myfile.parquet')

df = df[df.value >= 0]
joined = df.merge(other, on="account")
result = joined.groupby("account").value.mean()
```

Dask does pandas in parallel. Dask is lazy; when you want an in-memory result add `.compute()`. Persist data in distributed memory with the `.persist()` method. 

```python
import dask.dataframe as dd

df = dd.read_parquet('s3://mybucket/myfile.*.parquet')

df = df[df.value >= 0]
joined = df.merge(other, on="account")
result = joined.groupby("account").value.mean()

result.compute()
```

Dask comfortably scales to handle datasets that range in size from 10 GiB to 100 TiB. Dask DataFrame is used across a wide variety of applications—anywhere you're working with a large tabular dataset.

Here are a few large-scale examples using Coiled to deploy Dask on the cloud:

- [Parquet ETL with Dask DataFrame](https://docs.coiled.io/user_guide/uber-lyft.html)
- [XGBoost model training with Dask DataFrame](https://docs.coiled.io/user_guide/xgboost.html)
- [Visualize 1,000,000,000 points](https://docs.coiled.io/user_guide/datashader.html)

## Parallel For Loops

With Dask, you can parallelize any Python code, no matter how complex. Dask is flexible and supports arbitrary dependencies and fine-grained task scheduling that extends Python's concurrent.futures interface. Dask futures allow you to scale generic Python workflows across a Dask cluster with minimal code changes.

```python
from dask.distributed import LocalCluster

cluster = LocalCluster()                # Runs locally, but you can deploy Dask anywhere
client = cluster.get_client()


def process(filename):                  # Define your own Python function
   ...

futures = []
for filename in filenames:              # Submit many tasks in a loop
    future = client.submit(process, filename)
    futures.append(future)

results = client.gather(futures)        # Wait until done and gather results
```

As your data grows or your computational demands increase, you might want to deploy Dask on a cluster. If you're using the cloud, this is easy to do with Coiled:

```python
from coiled import Cluster

cluster = coiled.Cluster(n_workers=100)    # Request 100 VMs on AWS 
client = cluster.get_client()


def process(filename):                     # Define your own Python function
   ...

futures = []
for filename in filenames:                 # Submit many tasks in a loop
    future = client.submit(process, filename)
    futures.append(future)

results = client.gather(futures)           # Wait until done and gather results
```

Here's an [example extracting text](https://docs.coiled.io/user_guide/arxiv-matplotlib.html) from PDFs stored in a public S3 bucket, where we use Dask to scrape 1 TiB of data in ~5 minutes. 

## Big Xarray

Dask integrates well with Xarray to process array data in parallel. This includes common file formats like HDF, NetCDF, TIFF, or Zarr. Working with Dask and Xarray feels much like using NumPy arrays, but enables handling significantly larger datasets. The integration between Dask and Xarray is seamless, so you rarely need to manage parallelism explicitly—these details are handled automatically.

In the following example, we can import Zarr data and perform an aggregation without explicitly importing Dask at all. If Dask is available, it will be used automatically to process the data in parallel.

```python
import xarray as xr

ds = xr.open_zarr("/path/to/data.zarr")
timeseries = ds["temp"].mean(dim=["x", "y"]).compute()  # Compute result
```

This allows you to write scalable code that transitions from small, in-memory datasets on a single machine to large, distributed datasets on a cluster, requiring only minimal adjustments.

Here's an [example aggregating 250 TiB of raster data](https://docs.coiled.io/blog/coiled-xarray.html) from the NOAA National Water Model to calculate the average water table depth for each county in the US. 

## Dask for Machine Learning

Dask integrates with a number of machine learning libraries to train or predict on large datasets, increasing model accuracy by using *all* of your data.

Here's an illustrative example using Dask with XGBoost for distributed model training:

```python
import xgboost as xgb
import dask.dataframe as dd

df = dd.read_parquet("s3://my-data/")
dtrain = xgb.dask.DaskDMatrix(df)

model = xgb.dask.train(
    dtrain,
    {"tree_method": "hist", ...},
    ...
)
```

Model training can often be both computationally- and memory-intensive. Here are a few examples of speeding up machine learning tasks by taking advantage of cloud resources with Coiled:

- [Distributed model training with XGBoost + Dask](https://docs.coiled.io/user_guide/xgboost.html)
- [Model tuning with XGBoost + Optuna + Dask](https://docs.coiled.io/user_guide/hpo.html)
- [PyTorch model training on GPUs](https://docs.coiled.io/user_guide/pytorch.html)

## Deploying Dask

Many of these examples process larger-than-memory datasets on Dask clusters deployed on the cloud with Coiled. If your data is stored on the cloud or you need access to more/better hardware, you can use Coiled to run Dask on AWS, GCP, or Azure. See our documentation on all the ways you can [deploy Dask](https://docs.coiled.io/user_guide/dask-deployment-comparisons.html) for more details.

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/mfLyF7nZlRw?enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Dask on Single Machine with Coiled" data-gtm-yt-inspected-34277050_38="true" id="909946125" data-gtm-yt-inspected-12="true"></iframe>

‍