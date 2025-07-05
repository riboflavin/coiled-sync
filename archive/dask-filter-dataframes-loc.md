---
title: Filtering Dask DataFrames with loc
description: This post explains how to filter Dask DataFrames based on the DataFrame index and on column values using loc.
blogpost: true
date: 
author: 
---

# Filtering Dask DataFrames with loc

This post explains how to filter Dask DataFrames based on the DataFrame index and on column values using loc.

Filtering Dask DataFrames can cause data to be unbalanced across partitions which isn't desirable from a performance perspective. This post illustrates how filtering can cause the "empty partition problem" and how to eliminate empty partitions with repartitioning.

The blog starts with minimal complete verifiable examples ([MVCEs)](https://matthewrocklin.com/blog/work/2018/02/28/minimal-bug-reports) filtering examples on small datasets and then shows how to filter DataFrames with millions of rows of data on a cluster.

This data is stored in a public bucket and you can use Coiled for free if you'd like to run these computations yourself.

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kESVYngqDzLILC9Ff1xNrNDIy022Q&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=b8365e53-9392-4ee7-a2c9-b63c90d3db1e&amp;redirect_url=APefjpHZ3SBBS6euv4BPeJTyyT5M69agZK5-EhAVrkKpPWD1awKfKp4i61JcrpzfcJjrHPf6jNhmvy2I4OvlWKI4sPsubagt6DDuQNh3azArdeF-jNoZGdJB0PNtoTF3o0puhN9kNsAx&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fdask-filter-dataframes-loc&amp;ts=1744161886673" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span></div>

‍

Here's the [link to the notebook](https://github.com/coiled/coiled-resources/blob/master/filter-dataframes/filter-examples.ipynb) with all the code snippets used in this post.

## Dask Filter: Index filtering

Dask DataFrames consist of multiple partitions, each of which is a pandas DataFrame. Each pandas DataFrame has an index. Dask allows you to filter multiple pandas DataFrames on their index in parallel, which is quite fast.

Let's create a Dask DataFrame with 6 rows of data organized in two partitions.

```python
import pandas as pd
import dask.data
frame as dddf = pd.DataFrame({"nums":[1, 2, 3, 4, 5, 6], "letters":["a", "b", "c", "d", "e", "f"]})
ddf = dd.from_pandas(df, npartitions=2)
```

Let's visualize the data in each partition.

```python
for i in range(ddf.npartitions):
	print(ddf.partitions[i].compute())
	nums letters
0     1       a
1     2       b
2     3       c   
	nums letters
3     4       d
4     5       e
5     6       f
```

Dask automatically added an integer index column to our data.

Grab rows 2 and 5 from the DataFrame.

```python
ddf.loc[[2, 5]].compute()
	nums letters
2     3       c
5     6       f
```

Grab rows 3, 4, and 5 from the DataFrame.

```python
ddf.loc[3:5].compute()
	nums letters
3     4       d
4     5       e
5     6       f
```

Let's learn more about how Dask tracks information about divisions in sorted DataFrames to perform loc filtering efficiently.

## Dask Filter: Divisions

Dask is aware of the starting and ending index value for each partition in the DataFrame and stores this division's metadata to perform quick filtering.

You can verify that Dask is aware of the divisions for this particular DataFrame by running ddf.known_divisions and seeing it returns True. Dask isn't always aware of the DataFrame divisions.

Print all the divisions of the DataFrame.

```python
ddf.divisions(0, 3, 5)
```

Take a look at the values in each partition of the DataFrame to better understand this division's output.

```python
for i in range(ddf.npartitions):
	print(ddf.partitions[i].compute())  
	nums letters
0     1       a
1     2       b
2     3       c   
	nums letters
3     4       d
4     5       e
5     6       f
```

The first division is from 0-3, and the second division is from 3-5. This means the first division contains rows 0 to 2, and the last division contains rows 3 to 5.

Dask's division awareness in this example lets it know exactly what partitions it needs to fetch from when filtering.

‍

<div class="w-embed"><a class="blog-button" href="https://cloud.coiled.io/signup">Try Coiled</a></div>

‍

‍

## Dask Filter: Column value filtering

You won't always be able to filter based on index values. Sometimes you need to filter based on actual column values.

Fetch all rows in the DataFrame where nums is even.

```python
ddf.loc[ddf["nums"] % 2 == 0].compute()
    nums letters
    1   2   b
    3   4   d
    5   6   f
```

Find all rows where nums is even, and letters contains either b or f.

```python
ddf.loc[(ddf["nums"] % 2 == 0) & (ddf["letters"].isin(["b", "f"]))].compute()
    nums letters
    1   2   b
    5   6   f
```

Dask makes it easy to apply multiple logic conditions when filtering.

## Dask Filter: Empty partition problem

Let's read a dataset with 662 million rows into a Dask DataFrame and perform a filtering operation to illustrate the empty partition problem.

Read in the data and create the Dask DataFrame.

```python
ddf = dd.read_parquet(
    "s3://coiled-datasets/timeseries/20-years/parquet",
    storage_options={"anon": True, 'use_ssl': True})
```

ddf.npartitions shows that the DataFrame has 1,095 partitions.

Run ddf.head() to take a look at the data.

Here's how to filter the DataFrame to only include rows with an id greater than 1150.

```python
res = ddf.loc[ddf["id"] > 1150]
```

Run len(res) to see that the DataFrame only has 1,103 rows after this filtering operation. This was a big filter, and only a small fraction of the original 662 million rows remain.

We can run res.npartitions to see that the DataFrame still has 1,095 partitions. The filtering operation didn't change the number of partitions.

Run res.map_partitions(len).compute() to visually inspect how many rows of data are in each partition.

```python
0    0
1   1
2   0
3   0
4   2
    ..
1090    0
1091    2
1092    0
1093    0
1094    0
Length: 1095, dtype: int64
```

A lot of the partitions are empty and others only have a few rows of data.

Dask often works best with partition sizes of at least 100MB. Let's repartition our data to two partitions and persist it in memory.

res2 = res.repartition(2).persist()

Subsequent operations on res2 will be really fast because the data is stored in memory. len(res) takes 57 seconds whereas len(res2) only takes 0.3 seconds.

The filtered dataset is so small in this example that you could even convert it to a pandas DataFrame with res3 = res.compute(). It only takes 0.000011 seconds to execute len(res3).

You don't have to filter datasets at the computation engine level. You can also filter at the database level and only send a fraction of the data to the computation engine.

## Dask Filter: Query pushdown

Query pushdown is when you perform data operations before sending the data to the Dask cluster. Part of the work is "pushed down" to the database level.

[See the Advantages of Parquet blog post](https://coiled.io/blog/parquet-file-column-pruning-predicate-pushdown/) for information on column pruning and predicate pushdown filtering, query pushdown for the Parquet file format.

Here's the high-level process for filtering with a Dask cluster:

- Read all the data from disk into the cluster
- Perform the filtering operation
- Repartition the filtered DataFrame
- Possibly write the result to disk (ETL style workflow) or persist in memory

Organizations often need to optimize data storage and leverage query pushdown in a manner that's optimized for their query patterns and latency needs.

## Dask Filter: Conclusion

Dask makes it easy to filter DataFrames, but you need to be cognizant of the implications of big filters.

After filtering a lot of data, you should consider repartitioning and persisting the data in memory.

You should also consider filtering at the database level and bypassing cluster filtering altogether. Lots of Dask analyses run slower than they should because a large filtering operation was performed, and the analyst is running operations on a DataFrame with tons of empty partitions.

Thanks for reading. If you're interested in trying out Coiled, which provides hosted Dask clusters, docker-less managed software, and one-click deployments, you can do so for free today when you click below.

‍

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kESVYngqDzLILC9Ff1xNrNDIy022Q&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=b8365e53-9392-4ee7-a2c9-b63c90d3db1e&amp;redirect_url=APefjpHZ3SBBS6euv4BPeJTyyT5M69agZK5-EhAVrkKpPWD1awKfKp4i61JcrpzfcJjrHPf6jNhmvy2I4OvlWKI4sPsubagt6DDuQNh3azArdeF-jNoZGdJB0PNtoTF3o0puhN9kNsAx&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fdask-filter-dataframes-loc&amp;ts=1744161886673" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span></div>