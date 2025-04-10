---
title: How to Merge Dask DataFrames
description: This post demonstrates how to merge Dask DataFrames and discusses important considerations when making large joins.
blogpost: true
date: 
author: 
---

# How to Merge Dask DataFrames

<figure style="padding-bottom:56.25%" class="w-richtext-align-fullwidth w-richtext-figure-type-video"><div><iframe src="https://www.loom.com/embed/f30901c501ee442280b50f3d279b2e4f" allowfullscreen=" frameborder="0"></iframe></div></figure>

‍

This post demonstrates how to merge Dask DataFrames and discusses important considerations when making large joins.

You'll learn:

- how to join a large Dask DataFrame to a small pandas DataFrame
- how to join two large Dask DataFrames
- how to structure your joins for optimal performance

The lessons in this post will help you execute your data pipelines faster and more reliably, enabling you to deliver value to your clients in shorter cycles.

Let's start by diving right into the Python syntax and then build reproducible data science examples you can run on your machine.

‍

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-4db158e7-3b1a-49ab-872f-0f1a46cbbb7a">   <span class="hs-cta-node hs-cta-4db158e7-3b1a-49ab-872f-0f1a46cbbb7a" id="hs-cta-4db158e7-3b1a-49ab-872f-0f1a46cbbb7a">

    <a class="blog-button" style="margin-top: 16px; margin-bottom: 8px;" href="https://cta-redirect.hubspot.com/cta/redirect/9245528/4db158e7-3b1a-49ab-872f-0f1a46cbbb7a">DASK OFFICE HOURS</a>
</span>
</span></div>

‍

## Dask Dataframe Merge 

You can join a Dask DataFrame to a small pandas DataFrame by using the dask.dataframe.merge() method, similar to the pandas api.

Below we create a Dask DataFrame with multiple partitions and execute a left join with a small pandas DataFrame:

```python
import dask.dataframe as dd
import pandas as pd

# create sample large pandas dataframe
df_large = pd.DataFrame(
     {
        "Name": ["Azza", "Brandon", "Cedric", "Devonte", "Eli", "Fabio"], 
        "Age": [29, 30, 21, 57, 32, 19]
     }
)
# create multi-partition dask dataframe from pandas
large = dd.from_pandas(df_large, npartitions=2)

# create sample small pandas dataframe
small = pd.DataFrame(
     {
        "Name": ["Azza", "Cedric", "Fabio"], 
        "City": ["Beirut", "Dublin", "Rosario"]
     }
)

# merge dask dataframe to pandas dataframe
join = ddf.merge(df2, how="left", on=["Name"])

# inspect results
join.compute()//]]>
‍
```

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b60fbdbe1b42ece6904_9761-pasted-image-0.png)

To join two large Dask DataFrames, you can use the exact same Python syntax. If you are planning to run repeated joins against a large Dask DataFrame, it's best to sort the Dask DataFrame using the .set_index() method first to improve performance.

```python
large.set_index()

large_join = large.merge(also_large, how="left", left_index=True, right_index=True)
```

## Dask DataFrame merge to a small pandas DataFrame

Dask DataFrames are divided into multiple partitions. Each partition is a pandas DataFrame with its own index. Merging a Dask DataFrame to a pandas DataFrame is therefore an embarrassingly parallel problem. Each partition in the Dask DataFrame can be joined against the single small pandas DataFrame without incurring overhead relative to normal pandas joins.

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b60fbdbe16d14ce6902_9761-pasted-image-1.png)

‍

Let's demonstrate with a reproducible Python code example. Import dask.dataframe and pandas and then load in the datasets from the public Coiled Datasets S3 bucket. This time the Dask DataFrame is *actually* large and not just a placeholder: it contains a 35GB dataset of time series data. This means the data is too large to run with pandas on almost all machines.

```python
import dask.dataframe as dd
import pandas as pd 

# create dataframes
large = dd.read_parquet("s3://coiled-datasets/dask-merge/large.parquet")
small = pd.read_parquet("s3://coiled-datasets/dask-merge/small.parquet")

# inspect large dataframe
large.head()
```

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b60fbdbe14c55ce6905_9761-pasted-image-2.png)

//# check number of partitions
>>> large.npartitions
359

# inspect small dataframe
small.head()//]]>
‍

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b60fbdbe14f37ce68fc_9761-pasted-image-3.png)

The large dataframe contains a large dataset of synthetic time series data with entries at a frequency of 1 second. The small dataframe contains synthetic data over the same time interval but at a frequency of one entry per day.

Let's execute a left join on the timestamp column by calling dask.dataframe.merge(). We'll use large as the left dataframe.

//joined = large.merge(
     small, 
     how="left", 
     on=["timestamp"]
)

joined.head()//]]>

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b60fbdbe146cece6903_9761-pasted-image-4.png)

‍

As expected, the column z is filled with NaN for all entries except the first per-second entry of every day.

If you're working with a small Dask DataFrame instead of a pandas DataFrame, you have two options. You can [convert it into a pandas DataFrame](https://coiled.io/blog/converting-a-dask-dataframe-to-a-pandas-dataframe/) using .compute(). This will load the DataFrame into memory. Alternatively, if you can't or don't want to load it into your single machine memory, you can turn the small Dask DataFrame into a single partition by [using the .repartition() method](https://coiled.io/blog/repartition-dataframe/) instead. These two operations are programmatically equivalent which means there's no meaningful difference in performance between them. Rule of thumb here is to keep your Dask partitions under 100MB each.

//# turn dask dataframe into pandas dataframe
small = small.compute()

# OR turn dask dataframe into one partition
small = small.repartition(npartitions=1)//]]>

## Merge two large Dask DataFrames

You can merge two large Dask DataFrames with the same .merge() API syntax.

//large_joined = large.merge(
     also_large, 
     how="left", 
     on=["timestamp"]
)//]]>

However, merging two large Dask DataFrames requires careful consideration of your data structure and the final result you're interested in. Joins are expensive operations, especially in a distributed computing context. Understanding both your data and your desired end result can help you set up your computations efficiently to optimize performance. The most important consideration is whether and how to set your DataFrame's index before executing the join. Note that in the previous example, the timestamp column is the index for both dataframes.

### Unsorted vs Sorted Joins

As explained above, Dask DataFrames are divided into partitions, where each single partition is a pandas DataFrame. Dask can track how the data is partitioned (i.e. where one partition starts and the next begins) using a DataFrame's divisions.

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b60fbdbe1941ece6906_9761-pasted-image-5.png)

If a Dask DataFrame's divisions are known, then Dask knows the minimum value of every partition's index and the maximum value of the last partition's index. This enables Dask to take efficient shortcuts when looking up specific values. Instead of searching the entire dataset, it can find out *which partition* the value is in by looking at the divisions and then limit its search to only that *specific partition*. This is called a sorted join. The join stored in large_joined we executed above is an example of a sorted join since the timestamp column is the index for both of the dataframes in that join.

Let's look at another example. The divisions of the DataFrame df below are known: it has 4 divisions. This means that if we look up the row with index 2015-02-12, Dask will only search the 2nd partition and won't bother with the other three. In reality, Dask Dataframes often have hundreds or even thousands of partitions, which means the benefit of knowing where to look for a specific value becomes even greater.

//>>> df.known_divisions
True

>>> df.npartitions
4

>>> df.divisions
['2015-01-01', '2015-02-01', '2015-03-01', '2015-04-01', '2015-04-31']//]]>

If divisions are not known, then Dask will need to move all of your data around so that rows with matching values in the joining columns end up in the same partition. This is called an *unsorted* *join* and it's an extremely memory-intensive process, especially if your machine runs out of memory and Dask will have to read and write data to disk instead. This is a situation you want to avoid. Read more about unsorted large to large joins [in the Dask documentation](https://docs.dask.org/en/latest/dataframe-joins.html#large-to-large-unsorted-joins).

### Sorted Join with set index

To perform a sorted join of two large Dask DataFrames, you will need to ensure that the DataFrame's divisions are known by setting the DataFrame's index. You can set a Dask DataFrame's index and pass it the known divisions using:

//# use set index to get divisions
dask_divisions = large.set_index("id").divisions
unique_divisions = list(dict.fromkeys(list(dask_divisions)))

# apply set index to both dataframes
large_sorted = large.set_index("id", divisions=unique_divisions)
also_large_sorted = also_large.set_index("id", divisions=unique_divisions)

large_join = large_sorted.merge(
     also_large_sorted, 
     how="left", 
     left_index=True, 
     right_index=True
)//]]>

Note that setting the index is itself also an expensive operation. The rule of thumb here is that if you're going to be joining against a large Dask DataFrame *more than once*, it's a good idea to set that DataFrame's index first. Read our blog about [setting a Dask DataFrame index](https://coiled.io/blog/dask-set-index-dataframe/) to learn more about how and when to do this.

It's good practice to write sorted DataFrames to the Apache Parquet file format in order to preserve the index. If you're not familiar with Parquet, then you might want to check out our blog about [the advantages of Parquet for Dask analyses](https://coiled.io/blog/parquet-file-column-pruning-predicate-pushdown/).

### Joining along a non-Index column

You may find yourself in the situation of wanting to perform a join between two large Dask DataFrames along a column that is *not the index*. This is basically the same situation as not having set the index (i.e. an unsorted join) and will require a complete data shuffle, which is an expensive operation. Ideally you'll want to think about your computation in advance and set the index right from the start. The Open Source Engineering team at Coiled is working actively to improve shuffling. Read the [Proof of Concept for better shuffling in Dask if you'd like to learn more.](https://coiled.io/blog/better-shuffling-in-dask-a-proof-of-concept/)

### Sorted Join Fails Locally (MemoryError)

Even a sorted join may fail locally if the datasets are simply too large for your local machine. For example, this join:

//large = dd.read_parquet("s3://coiled-datasets/dask-merge/large.parquet")
also_large = dask.datasets.timeseries(start="1990-01-01", end="2020-01-01", freq="1s", partition_freq="1M", dtypes={"foo": int})

large_join = large.merge(
     also_large, 
     how="left", 
     left_index=True, 
     right_index=True
)

large_join.persist()//]]>

Will throw the following error when run on a laptop with 32GB of RAM or less.

[ error ]

*Note that we didn't set the index explicitly here because the index of large is preserved in the Parquet file format and dask.datasets.timeseries() automatically sets the in*dex when creating the synthetic data.

### Run Massive Joins on Dask Cluster 

When this happens, you can scale out to a Dask cluster in the cloud with Coiled and run the join there in 3 steps:

1. Spin up a Coiled cluster:

//cluster = coiled.Cluster(
     name="dask-merge",
     n_workers=50,
     worker_memory='16Gib',
     backend_options={'spot':'True'},
)//]]>

2. Connect Dask to the running cluster:

//from distributed import Client
client = Client(cluster)//]]>

3. Run the massive join on 50 workers' multiple cores in parallel:

//large_join = large.merge(
     also_large, 
     how="left", 
     left_index=True, 
     right_index=True
)

%%time
joined = large_join.persist()
distributed.wait(joined)

CPU times: user 385 ms, sys: 38.5 ms, total: 424 ms
Wall time: 14.6 s//]]>

This Coiled cluster with 50 workers can run a join of two 35GB DataFrames in 14.6s.

## Dask Dataframe Merge Summary

- You can merge a Dask DataFrame to a small pandas DataFrame using the merge method. This is an embarrassingly parallel problem that requires little to no extra overhead compared to a regular pandas join.
- You can merge two large Dask DataFrames using the same merge method. Think carefully about whether to run an unsorted join or a sorted join using set_index first to speed up the join.
- For very large joins, delegate computations to a Dask cluster in the cloud with Coiled for high-performance joins using parallel computing.

## Do you have more Dask questions?

Join us in the Dask Office Hours by Coiled.

‍

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-4db158e7-3b1a-49ab-872f-0f1a46cbbb7a">   <span class="hs-cta-node hs-cta-4db158e7-3b1a-49ab-872f-0f1a46cbbb7a" id="hs-cta-4db158e7-3b1a-49ab-872f-0f1a46cbbb7a">

    <a class="blog-button" style="margin-top: 16px; margin-bottom: 8px;" href="https://cta-redirect.hubspot.com/cta/redirect/9245528/4db158e7-3b1a-49ab-872f-0f1a46cbbb7a">DASK OFFICE HOURS</a>
</span>
</span></div>