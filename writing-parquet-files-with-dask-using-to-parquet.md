---
title: Writing Parquet Files with Dask using to_parquet
description: This blog post explains how to write Parquet files with Dask using the to_parquet method.
blogpost: true
date: 
author: 
---

# Writing Parquet Files with Dask using to_parquet

This blog post explains how to write Parquet files with Dask using the to_parquet method.

The Parquet file format allows users to enjoy several performance optimizations when reading data, benefiting downstream users of the data.

This blog post will teach you the basics of writing Parquet files and advanced options like customizing the filenames and controlling if the metadata file gets written.  You need to be careful to avoid writing the metadata file when it can become a performance bottleneck.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kGGoXG9KjRfeIb2xMmXbQWrVF_zZw&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=54d84fa1-3535-4d91-b312-5fc3d78943f4&amp;redirect_url=APefjpHGo0QiZ4dGXNms4W9yVd8DebTUu5m_LaYco7jsOL2moU2twey0p99du-iFOFuSjCPVEl3LdE01U-qBzAzJsMBR2goG9IUfjceF2HHw21EuXXf4eohn2YWabkNZmgmMHx-Ty1EG&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fwriting-parquet-files-with-dask-using-to-parquet&amp;ts=1744162586401" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>

Let's dive in with a simple example.

## Dask write Parquet: Small example

Let's create a small Dask DataFrame and then write it out to disk with to_parquet.

```python
import dask.dataframe as dd
import pandas as pd

df = pd.DataFrame(
    {"nums": [1, 2, 3, 4, 5, 6], "letters": ["a", "b", "c", "d", "e", "f"]}
)
ddf = dd.from_pandas(df, npartitions=2)

ddf.to_parquet("data/something", engine="pyarrow")
```

Here are the files that are output to disk.

```python
data/something/
  _common_metadata
  _metadata
  part.0.parquet
  part.1.parquet
```

Dask DataFrame's file I/O writer methods output one file per partition.  This DataFrame has two partitions, so two Parquet files are written.  The Parquet writer also outputs _common_metadata and _metadata files by default.

Parquet files contain metadata about the file contents including the schema of the data, the column names, the data types, and min/max values for every column in each row group.  This metadata is part of what allows for Parquet files to be read in a more performant manner compared to file formats like CSV.

Dask writes out the metadata in the Parquet file footers to a single file to bypass the need to fetch the metadata from each Parquet file that's being read.  Footers are a section of Parquet files where metadata is stored.  It can just look at the metadata file to get the metadata for each file in the directory.  Creating the metadata file is great for small datasets, but can become a performance bottleneck when lots of files are written.

## Dask to_parquet: write_metadata_file

The write_metadata_file argument is set to True by default.  For large writes, it's good to set write_metadata_file to False, so collecting all the Parquet metadata in a single file is not a performance bottleneck.

Here's how to write out the same data as before, but without a _metadata file.

```python
ddf.to_parquet(
    "data/something2",
    engine="pyarrow",
    write_metadata_file=False,
)
```

Here are the files that are output to disk.

```python
data/something2/
  part.0.parquet
  part.1.parquet
```

When Dask reads a folder with Parquet files that do not have a _metadata file, then Dask needs to read the file footers of all the individual Parquet files to gather the statistics.  This is slower than reading the data directly from a _metadata file.  But, as mentioned previously, trying to write too much data to a single _metadata file can be prohibitively slow when lots of files are written.

In short, set write_metadata_file to False when doing large Parquet writes.

## Dask to_parquet: name_function

The Parquet writes will output files with names like part.0.parquet and part.1.parquet by default.  You can set the name_function parameter to customize the filenames that are written with to_parquet.

Let's output a universally unique identifier (UUID) in the filename that's output so we can see what files are written per each batch.

```python
import uuid

id = uuid.uuid4()
def batch_id(n):
    return f"part-{n}-{id}.parquet"

ddf.to_parquet(
    "data/something3",
    engine="pyarrow",
    write_metadata_file=False,
    name_function=batch_id,
)
```

Here are the files that are output to disk.

```python
data/something3/
part-0-09a19442-309e-485f-b006-fd9cb10f9cc7.parquet
part-1-09a19442-309e-485f-b006-fd9cb10f9cc7.parquet
```

It's really nice to write files like this in your production applications.  This lets you easily identify the files that are written, per batch.

You can also use the name_function to write out files with a timestamp, so they're ordered chronologically.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kGGoXG9KjRfeIb2xMmXbQWrVF_zZw&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=54d84fa1-3535-4d91-b312-5fc3d78943f4&amp;redirect_url=APefjpHGo0QiZ4dGXNms4W9yVd8DebTUu5m_LaYco7jsOL2moU2twey0p99du-iFOFuSjCPVEl3LdE01U-qBzAzJsMBR2goG9IUfjceF2HHw21EuXXf4eohn2YWabkNZmgmMHx-Ty1EG&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fwriting-parquet-files-with-dask-using-to-parquet&amp;ts=1744162586401" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>

## Dask to_parquet: compression

Dask writes out Parquet files with Snappy compression by default.  Snappy compression is typically the best for files used in distributed compute contexts.  Snappy doesn't usually compress files as much as other compression algorithms like gzip, but it's faster when decompressing files (aka inflating files).  You'll often read compressed files from disk in big data systems, so fast inflation is important.

You can specify the Snappy compression algorithm, but this doesn't change anything because Snappy is used by default.

```python
ddf.to_parquet(
    "data/something4",
    engine="pyarrow",
    write_metadata_file=False,
    compression="snappy",
)
```

You can write out the files with the compression algorithm in the filename, which can be convenient.

```python
def with_snappy(n):
    return f"part-{n}.snappy.parquet"

ddf.to_parquet(
    "data/something5",
    engine="pyarrow",
    write_metadata_file=False,
    compression="snappy",
    name_function=with_snappy,
)
```

Here are the files that are output to disk.

```python
data/something5/
  part-0.snappy.parquet
  part-1.snappy.parquet
```

This is a nice way to make it clear to other humans what compression algorithm is being used by the file. Note that you will need to manually update the name_function parameter to reflect any changes made to the compression parameter.

Different columns of a Parquet file can actually be compressed with different compression algorithms.  Here's how to compress the nums column with Snappy and the letters column with gzip.

```python
ddf.to_parquet(
    "data/something6",
    engine="pyarrow",
    write_metadata_file=False,
    compression={"nums": "snappy", "letters": "gzip"},
)
```

You don't normally see different compression algorithms for different columns in a Parquet file, but it's a cool feature that's incredibly handy for niche use cases.

## Dask write parquet: FastParquet vs PyArrow Engines

You can write Parquet files with both the FastParquet and PyArrow engines.  For most use cases, either selection is fine.  There are some subtle differences between the two engines, but they don't matter for most users.

PyArrow's popularity has taken off and it's well supported in Dask, so PyArrow is a good option if you're not sure which engine to use.

## Dask write parquet: partition_on

You can write Parquet files in a Hive-compliant directory structure that allows for readers to leverage disk partition filtering by setting the partition_on argument.

Disk partition filtering can be a significant performance optimization for certain types of queries, but it will make other types of queries run slower.  For example, queries that filter or group-by on the partition key will run a lot faster than queries operating on different keys.  Queries that don't operate on the partition key will face the drag of extra files and globbing nested directories.  Globbing is particularly expensive in cloud based object stores like AWS S3.

See the post on [Creating Dask Partitioned Lakes with Dask using partition_on](https://coiled.io/blog/dask-disk-partition-on/) for more information.  Disk partitioning allows for huge performance gains for some query patterns, so you should know how it works.

## Dask to_parquet Conclusion

Dask makes it easy to write Parquet files and provides several options allowing you to customize the operation.

You can specify the compression algorithm, choose if the metadata file should be written, customize the filename, and write the files in a nested directory structure for disk partition filtering.

Consider omitting the metadata file when you must write a lot of partitions because it can be a performance bottleneck.  Including a UUID and the compression algorithm in the filename is nice when you're building production pipelines that repeatedly write to the same directory.

You can always repartition before writing to change the number of files that are written.  See [this blog post](https://coiled.io/blog/repartition-dataframe/) for more information about the repartition method.

See [this blog post](https://coiled.io/blog/parquet-file-column-pruning-predicate-pushdown/) to learn more about the performance benefits you can get from column pruning and predicate pushdown filtering, two optimization techniques that are afforded by Parquet files but not available in other file formats like CSV or JSON.

If you'd like to scale your Dask work to the cloud, check out Coiled — Coiled provides quick and on-demand Dask clusters along with tools to manage environments, teams, and costs. Click below to learn more!

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kGGoXG9KjRfeIb2xMmXbQWrVF_zZw&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=54d84fa1-3535-4d91-b312-5fc3d78943f4&amp;redirect_url=APefjpHGo0QiZ4dGXNms4W9yVd8DebTUu5m_LaYco7jsOL2moU2twey0p99du-iFOFuSjCPVEl3LdE01U-qBzAzJsMBR2goG9IUfjceF2HHw21EuXXf4eohn2YWabkNZmgmMHx-Ty1EG&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fwriting-parquet-files-with-dask-using-to-parquet&amp;ts=1744162586401" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More