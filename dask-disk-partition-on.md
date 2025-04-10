---
title: Creating Disk Partitioned Lakes with Dask using partition_on
description: This post explains how to create disk partitioned Parquet lakes with Dask using partition_on. It also explains how to read disk partitioned lakes with read_parquet and how this can improve query speeds.
blogpost: true
date: 
author: 
---

# Creating Disk Partitioned Lakes with Dask using partition_on

This post explains how to create disk-partitioned Parquet lakes using `partition_on` and how to read disk-partitioned lakes with `read_parquet` and `filters`. Disk partitioning can significantly improve performance when used correctly. 

However, when used naively disk-partitioning may in fact reduce performance (because of expensive file listing operations or lots of small files). This article will build your intuition and skills around recognizing and implementing opportunities to improve performance using `partition_on`.

Let's start by outputting a Dask DataFrame with `to_parquet` using disk partitioning to develop some intuition on how this design pattern can improve query performance.

## Output disk partitioned lake

Create a Dask DataFrame with `letter` and `number` columns and write it out to disk, partitioning on the letter column. 

```python
df = pd.DataFrame(
    {"letter": ["a", "b", "c", "a", "a", "d"], "number": [1, 2, 3, 4, 5, 6]}
)
ddf = dd.from_pandas(df, npartitions=3)
ddf.to_parquet("tmp/partition/1", engine="pyarrow", partition_on="letter")
```

Here are the files that are written to disk.

```python
tmp/partition/1/
  letter=a/
    part.0.parquet
    part.1.parquet
    part.2.parquet
  letter=b/
    part.0.parquet
  letter=c/
    part.1.parquet
  letter=d/
    part.2.parquet
```

Organizing the data in this directory structure lets you easily skip files for certain read operations. For example, if you only want the data where the `letter` equals `a`, then you can look in the `tmp/partition/1/letter=a` directory and skip the other Parquet files.

The `letter` column is referred to as the partition key in this example.

Let's take a look at the syntax for reading the data from a certain partition.

## Read disk partitioned lake

Here's how to read the data in the `letter=a` disk partition.

```python
ddf = dd.read_parquet(
    "tmp/partition/1", engine="pyarrow", filters=[("letter", "==", "a")]
)
print(ddf.compute())

   number letter
0       1      a
3       4      a
4       5      a
```

Dask is smart enough to apply the partition filtering based on the filters argument. Dask only reads the data from tmp/partition/1/letters=a into the DataFrame and skips all the files in other partitions.

## Trade Offs

Disk partitioning improves performance for queries that filter on the partition key. It usually hurts query performance for queries that don't filter on the partition key.

*Example query that runs faster on disk partitioned lake*

```python
dd.read_parquet(
    "tmp/partition/1", engine="pyarrow", filters=[("letter", "==", "a")]
)
```

*Example query that runs slower on disk partitioned lake*

```python
ddf = dd.read_parquet("tmp/partition/1", engine="pyarrow")
ddf = ddf.loc[ddf["number"] == 2]
print(ddf.compute())
```

The performance drag from querying a partitioned lake without filtering on the partition key depends on the underlying filesystem. Unix-like filesystems are good at performing file listing operations on nested directories. So you might not notice much of a performance drag on your local machine.

Cloud-based object stores, like AWS S3, are not Unix-like filesystems. They store data as key value pairs and are slow when listing nested files with a wildcard character (aka globbing).

You need to carefully consider your organization's query patterns when evaluating the costs/benefits of disk partitioning. If you are always filtering on the partition key, then a partitioned lake is typically best. Disk partitioning might not be worth it if you only filter on the partitioned key sometimes.

## Multiple partition keys

You can also partition on multiple keys (the examples thus far have only partitioned on a single key).

Create another Dask DataFrame and write it out to disk with multiple partition keys.

```python
df = pd.DataFrame(
    [
        ["north america", "mexico", "carlos"],
        ["asia", "india", "ram"],
        ["asia", "china", "li"],
    ],
    columns=["continent", "country", "first_name"],
)
ddf = dd.from_pandas(df, npartitions=2)
ddf.to_parquet(
    "tmp/partition/2", engine="pyarrow", partition_on=["continent", "country"]
)
```

Here are the files that are outputted to the filesystem:

```python
tmp/partition/2/
  continent=asia/
    country=china/
      part.0.parquet
    country=india/
      part.0.parquet
  continent=north america/
    country=mexico/
      part.0.parquet
```

Read in the files where `continent=asia` and `country=china`.

```python
ddf = dd.read_parquet(
    "tmp/partition/2",
    engine="pyarrow",
    filters=[("continent", "==", "asia"), ("country", "==", "china")],
)
print(ddf.compute())

  first_name continent country
2         li      asia   china
```

Deeply partitioned lakes are even more susceptible to the partitioned lake performance pitfalls because file listing operations on cloud stores for nested folders can be slow. You'll typically only be rewarded with performance gains when you filter on at least one of the partition keys.

Globbing gets more expensive as the directory structure gets more nested.

## Additional considerations

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5c64f4f9e65bf53417_slack-imgs-700x368.png" alt=">

Key Point

When creating a partitioned lake, you need to choose a partition key that doesn't have too many distinct values.

Suppose you have a 5GB dataset and often filter on `column_a`. Should you partition your data lake on column_a?

That depends on the number of unique values in `column_a`. If `column_a` has 5 unique values, then it could be a great partition key. If `column_a` has 100,000 distinct values, then partitioning the 5GB dataset will break up the dataset into at least 100,000 files, which is way too much. That'd be 100,000 files that are only 0.05MB and it's usually best to target files that are 100MB, see [this blog post](https://coiled.io/blog/repartition-dataframe/) for more detail.

Partitioned data lakes are particularly susceptible to the small file problem. Dask analyses often work best with files that are around 100 MB when uncompressed. A data lake that's incrementally updated frequently will often create files that are quite small. Incrementally updating partitioned lakes will create files that are even smaller.

Suppose you have a data pipeline that ingests 600 MB of data per hour. If you're writing this data to an unpartitioned lake, then you can repartition into 100MB partitions and write out 6 files per hour. If you're writing to a partitioned lake with 10,000 unique partition key values, then you may be writing up to 10,000 files per hour.

Make sure you have a small file compaction routine if you build a lake that's partitioned on a high cardinality partition key and incrementally update frequently.


## Conclusion

Disk partitioning is a powerful performance optimization that can make certain queries much faster.

Due to the key-value nature of cloud object stores, disk partitioning can also make certain queries a lot slower.

It's a powerful optimization, but you need to understand your query patterns and limitations to make it work well with your data systems.

Thanks for reading! And if you're interested in trying out Coiled Cloud, which provides hosted Dask clusters, docker-less managed software, and one-click deployments, you can do so for free today when you click below.


<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_be9599a4-6c30-4743-aa64-3e10e4ddc6b6" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=be9599a4-6c30-4743-aa64-3e10e4ddc6b6&amp;signature=AAH58kHPbSh5m1kBoSCEyLufSMCvhiejhA&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=13fd439d-7d4a-4c30-90d7-cb6363a35157&amp;redirect_url=APefjpHYntfv-OjOz91hS9UJunvLyZOVXxAe2PJlOFY98nDx2JYS4SP5cYnL54QdAecCoMrQdLlblanWWwyCNAPUApbmQc9857u6HAD-lRh_i8S39aYPBGM&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fdask-disk-partition-on&amp;ts=1744161549380" style=" cta_dest_link="https://cloud.coiled.io/signup" title="Try Coiled"><div style="text-align: center;"><span style="font-family: Helvetica, Arial, sans-serif;"><span style="font-size: 16px;">Try </span><span style="font-size: 16px;">Coiled</span></span></div></a></span>
</span>