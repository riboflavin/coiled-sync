---
title: Speed up a pandas query 10x with these 6 Dask DataFrame tricks
description: This post demonstrates how to speed up a pandas query to run 10 times faster with Dask using six performance optimizations.
blogpost: true
date: 
author: 
---

# Speed up a pandas query 10x with these 6 Dask DataFrame tricks

***Updated April 18th, 2024***: *For Dask versions >= 2024.3.0, Dask will perform a number of the optimizations discussed in this blog post automatically. See [this demo](https://youtu.be/HTKzEDa2GA8?si=FkYToNKW7ooBQ-bA&t=38) for more details.*

This post demonstrates how to speed up a pandas query to run 10 times faster with Dask using six performance optimizations. You'll often want to incorporate these tricks into your workflows so your data analyses can run faster. 

After learning how to optimize the pandas queries on your local machine, you'll see how to query much larger datasets with cloud resources using Dask clusters that are managed by Coiled.

Here are the 6 strategies covered in this post:

- Parallelize the query with Dask
- Avoid `object` columns

- Split up single large file into 100 MB files
- Use Parquet instead of CSV
- Compress files with Snappy
- Leverage column pruning

This diagram illustrates how the query time decreases as each performance optimization is applied.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b603a4f10fc6f5db5b1_Screen-Shot-2022-02-11-at-9.12.15-AM-700x425.png" alt=">

After seeing how to optimize this query on a 5 GB dataset, we'll check out how to run the same query on a 50 GB dataset with a Dask cluster.  The cluster computation will show that these performance optimizations also help when querying large datasets in the cloud.

Let's start by running a groupby operation on a 5 GB dataset with pandas to establish a runtime baseline.

## Pandas query baselines

The queries in this post will be run on a 5 GB dataset that contains 9 columns. Here are the first five rows of data, so you can get a feel of the data contents.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b603a4f10ccb25db5b0_9795-Screen-Shot-2022-01-25-at-6.01.17-AM.png" alt=">

‍

You can run this AWS CLI command to download the dataset.

**aws s3 cp s3://coiled-datasets/h2o-benchmark/N_1e8_K_1e2_single.csv data/**

Let's use pandas to run a groupby computation and establish a performance baseline.

```python
import pandas as pd

df = pd.read_csv("data/N_1e8_K_1e2_single
.csv")
df.groupby("id1", dropna=False, observed=True).agg({"v1": "sum"})
```

This query takes 182 seconds to run.

Here's the query result:

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b603a4f1003945db5b2_9795-Screen-Shot-2022-01-25-at-6.14.36-AM.png" alt=">

‍

Let's see how Dask can make this query run faster, even when the Dask computation is not well structured.

If you don't want to download this dataset locally, you can also run these computations on the cloud. Later in this blog post, we'll provide a demo that'll show you how to use Coiled to easily spin up a Dask cluster and run these computations with machines in the cloud.

## Dask query baseline

Let's run the same groupby query with Dask. We're going to intentionally type all the columns as object columns, which should give us a good worst case Dask performance benchmark. object columns are notoriously memory hungry and inefficient.

```python
dtypes = {
    "id1": "object",
    "id2": "object",
    "id3": "object",
    "id4": "object",
    "id5": "object",
    "id6": "object",
    "v1": "object",
    "v2": "object",
    "v3": "object",
}

import dask.dataframe as dd

ddf = dd.read_csv("data/N_1e8_K_1e2_single.csv", dtype=dtypes)
```

Let's type v1 as an integer column and then run the same groupby query as before.

```python
ddf["v1"] = ddf["v1"].astype("int64")

ddf.groupby("id1", dropna=False, observed=True).agg({"v1": "sum"}).compute()
```

That query runs in 122 seconds.

Dask runs faster than pandas for this query, even when the most inefficient column type is used, because it parallelizes the computations. pandas only uses 1 CPU core to run the query. My computer has 4 cores and Dask uses all the cores to run the computation.

Let's recreate the DataFrame with more efficient data types and see how that improves the query runtime.

## Avoiding object columns with Dask

We can type id1, id2, and id3 as string type columns, which are more efficient as described in the following video.

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/RAAcDkQgBTI?start=364&enablejsapi=1&origin=https%3A%2F%2Fwww.coiled.io" title="Dask DataFrame is Fast Now" data-gtm-yt-inspected-11="true" id="148308292" data-gtm-yt-inspected-34277050_38="true"></iframe>

‍

id4, id5, id6, v1, and v2 can be typed as integers. v3 can be typed as a floating point number. Here is the revised computation.

```python
better_dtypes = {
    "id1": "string[pyarrow]",
    "id2": "string[pyarrow]",
    "id3": "string[pyarrow]",
    "id4": "int64",
    "id5": "int64",
    "id6": "int64",
    "v1": "int64",
    "v2": "int64",
    "v3": "float64",
}

ddf = dd.read_csv("data/N_1e8_K_1e2_single.csv", dtype=better_dtypes)

ddf.groupby("id1", dropna=False, observed=True).agg({"v1": "sum"}).compute()
```

This query runs in 67 seconds. Avoiding object type columns allows for a significant performance boost.

## Split data in multiple files

Let's split up the data into multiple files instead of a single 5 GB CSV file. Here's code that'll split up the data into 100 MB CSV files.

```python
ddf.repartition(partition_size="100MB").to_csv("data/csvs")
```

Let's rerun the query on the smaller CSV files.

```python
ddf = dd.read_csv("data/csvs/*.part", dtype=better_dtypes)

ddf.groupby("id1", dropna=False, observed=True).agg({"v1": "sum"}).compute()
```

The query now runs in 61 seconds, which is a bit faster. Dask can read and write multiple files in parallel. Parallel I/O is normally a lot faster than working with a single large file.

Let's try to use the Parquet file format and see if that helps.

## Use Parquet instead of CSV

Here's the code that'll convert the small CSV files into small Parquet files.

```python
ddf.to_parquet("data/parquet", engine="pyarrow", compression=None)
```

Let's rerun the same query off the Parquet files.

```python
ddf = dd.read_parquet("data/parquet", engine="pyarrow")

ddf.groupby("id1", dropna=False, observed=True).agg({"v1": "sum"}).compute()
```

This query executes in 39 seconds, so Parquet provides a nice performance boost. Columnar file formats that are stored as binary usually perform better than row-based, text file formats like CSV. Compressing the files to create smaller file sizes also helps.

## Compress Parquet files with Snappy

Let's recreate the Parquet files with Snappy compression and see if that helps.

```python
ddf.to_parquet("data/snappy-parquet", engine="pyarrow", compression="snappy")
```

Let's rerun the same query off the Snappy compressed Parquet files.

```python
ddf = dd.read_parquet("data/snappy-parquet", engine="pyarrow")

ddf.groupby("id1", dropna=False, observed=True).agg({"v1": "sum"}).compute()
```

This query runs in 38 seconds, which is a bit of a performance boost.

In this case, the files are stored on my hard drive and are relatively small. When data is stored in cloud based object stores and sent over the wire, file compression can result in an even bigger performance boost.

Let's leverage the columnar nature of the Parquet file format to make the query even faster.

## Leverage column pruning

Parquet is a columnar file format, which means you can selectively grab certain columns from the file. This is commonly referred to as column pruning. Column pruning isn't possible for row based file formats like CSV.

Let's grab the columns that are relevant for our query and run the query again.

```python
ddf = dd.read_parquet("data/snappy-parquet", engine="pyarrow", columns=["id1", "v1"])

ddf.groupby("id1", dropna=False, observed=True).agg({"v1": "sum"}).compute()
```

This query runs in 19 seconds.

## Overall performance improvement

The original pandas query took 182 seconds and the optimized Dask query took 19 seconds, which is about 10 times faster.

Dask can provide performance boosts over pandas because it can execute common operations in parallel, where pandas is limited to a single core. For this example, I configured Dask to use the available hardware resources on my laptop to execute the query faster. Dask scales with your hardware to optimize performance.

My machine only has 4 cores and Dask already gives great results. A laptop that has 14 cores would be able to run the Dask operations even faster.

Parquet files allow this query to run a lot faster than CSV files. The query speeds get even better when Snappy compression and column pruning are used.

The query optimization patterns outlined in this blog are applicable to a wide variety of use cases. Splitting your data into optimally sized files on disk makes it easier for Dask to read the files in parallel easier. Strategically leveraging file compression and column pruning almost always helps. There are often easy tweaks that'll let you run your pandas code a lot faster with Dask.

## Querying a large dataset with the cloud

Let's take a look at how to run this same query on a 50 GB dataset that's stored in the cloud.

Downloading the 50 GB dataset to our laptop and querying it in a localhost environment would be tedious and slow.  You're limited to the computing power of your laptop when running analysis locally.

Dask is designed to either be run on a laptop or with a cluster of computers that process the data in parallel.  Your laptop may only have 8GB or 32GB of RAM, so its computation power is limited.  Cloud clusters can be constructed with as many workers as you'd like, so they can be made quite powerful.

Cloud devops is challenging, so let's use Coiled to build our Dask cluster.  This will make it a lot easier than building a production grade Dask solution ourselves.

Here's how to make a Dask cluster with 15 nodes (each with 16 GiB of RAM) to query the 50 GB dataset.

Here's how to spin up the Dask cluster with Coiled:

```python
import coiled

cluster = coiled.Cluster(
    n_workers=15,                     
    region="us-east-2",               # run in the same region as data
    spot_policy="spot_with_fallback", # save money with spot instances
)

client = cluster.get_client()
```

Each instance has 16 GiB of RAM.  The cluster has 15 nodes, so the entire cluster has 240 GiB of RAM.  That's a lot of computing resources to perform an analysis on a large dataset!

Once the cluster is created, we can read in data from S3 into a Dask DataFrame.

```python
ddf = dd.read_parquet(
    "s3://coiled-datasets/h2o-benchmark/N_1e9_K_1e2_parquet",
    storage_options={"anon": True, "use_ssl": True},
    engine="pyarrow",
)
```

Let's see how many rows of data the DataFrame contains.

```python
len(ddf) # 1000000000
```

Wow, this DataFrame contains a billion rows of data.  Let's run the same query as before and see how long it takes:

```python
ddf.groupby("id1", dropna=False, observed=True).agg({"v1": "sum"}).compute()
```

The query runs in 10 seconds. We can see that Coiled allows us to easily spin up a Dask cluster and query a large dataset quickly. It's free and easy to [get started with Coiled](https://docs.coiled.io/user_guide/setup/index.html) and run the query yourself.

Part of the reason this query is running so fast is because we're using the query optimizations covered earlier.  We're querying snappy compressed Parquet files that each contain 100 MB of data and Dask DataFrame automatically uses column pruning to grab the relevant columns for the query.

## Conclusion

pandas is a fine technology for smaller datasets, but when the data volume is large, you'll want to use a cluster computing technology like Dask that can scale beyond the limits of a single machine.

You can manually create a Dask cluster yourself, but it's a lot of devops work.  Most data scientists and data engineers don't want to be bothered with a challenging devops project like productionalizing and tuning a Dask environment.  It's generally best to use Coiled, a service that lets you focus on your analysis without worrying about Dask devops.

In this post you've seen how to significantly speed up a pandas query and you've also taken a glimpse of harnessing the power of the cloud to run a computation in a cluster.  These tactics will serve you well when you're running analyses on larger datasets.