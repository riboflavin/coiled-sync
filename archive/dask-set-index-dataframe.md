---
title: Setting a Dask DataFrame index
description: This post demonstrates how to change a DataFrame index with sebt_index and explains when you should perform this operation.
blogpost: true
date: 
author: 
---

# Setting a Dask DataFrame index

This post demonstrates how to change a DataFrame index with set_index and explains when you should perform this operation.

Setting an index can make downstream queries faster, but usually requires an expensive data shuffle, so it should be used strategically.

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_be9599a4-6c30-4743-aa64-3e10e4ddc6b6" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=be9599a4-6c30-4743-aa64-3e10e4ddc6b6&amp;signature=AAH58kHGX-5oDnf7-w1PgaJP8-8KRs0g9g&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=24978215-edb6-419d-8bbf-c721d11a9e55&amp;redirect_url=APefjpGygs2XfX8Ubi3nFEK7U7pBynqySrj2lvFeQUPuREabPSJaFVysiiXNUFqCtFfWTTd0e-R0WYJIbFKuzNpdhABo7C8PVVKdMUkRO5mgvDyZyGaV5zQ&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fdask-set-index-dataframe&amp;ts=1744161887615" style=" cta_dest_link="https://cloud.coiled.io/signup" title="Try Coiled"><div style="text-align: center;"><span style="font-family: Helvetica, Arial, sans-serif;"><span style="font-size: 16px;">Try </span><span style="font-size: 16px;">Coiled</span></span></div></a></span>
</span></div>

This post demonstrates the set_index syntax on small datasets and then shows some query benchmarks on larger datasets.

## What is an index?

Indexes are used by normal pandas DataFrames, Dask DataFrames, and many databases in general.  

Indexes let you efficiently find rows that have a certain value, without having to scan each row. In plain pandas, it means that after a set_index("col"), df.loc["foo"] is faster than df[df.col == "foo"] was before. The loc uses an efficient data structure that only has to check a couple rows to figure out where "foo" is. Whereas df[df.col == "foo"] has to scan every single row to see which ones match.

The thing is, computers are really really fast at scanning memory, so when you're running pandas computations on one machine, index optimizations aren't as important. 

But scanning memory across many distributed machines is not fast. So index optimizations that you don't notice much with pandas makes an enormous difference with Dask.

A Dask DataFrame is composed of many pandas DataFrames, potentially living on different machines, each one of which we call a *partition*. See the Dask docs for an [illustration](https://docs.dask.org/en/stable/dataframe.html#design) and [in-depth explanation](https://docs.dask.org/en/stable/dataframe-design.html#partitions).

The Dask client has its own version of an index for the distributed DataFrame as a whole, called divisions. divisions is like an index for the indexes—it tracks which *partition* will contain a given value (just like pandas's index tracks which *row* will contain a given value). When there are millions of rows spread across hundreds of machines, it's too much to track every single row. We just want to get in the right ballpark—which machine will hold this value?—and then tell that machine to go find the row itself.

So divisions is just a simple list giving the lower and upper bounds of values that each partition contains. Using this, Dask does a quick binary search locally to figure out which partition contains a given value.

Just like with a pandas index, having known divisions let us change a search that would scan every row (df[df.col == "foo"]) to one that quickly goes straight to the right place (df.loc["foo"]). It's just that scanning every row is much, much slower in a distributed context with Dask than on one machine with plain pandas.

## When to set an index

Setting an index in Dask is a lot slower than setting an index in pandas.  You shouldn't always set indexes in Dask like you do in pandas. 

In pandas, you can get away with calling set_index even when it's not necessary. Sometimes, you'll see pandas code like set_index("foo").reset_index() in the wild, which sorts the data for no reason. In pandas, you can get away with this, because it's cheap. In Dask it's very, very not cheap. So you really should understand what set_index does and why you're doing it before you use it.

## set_index syntax

Let's create a small DataFrame and set one of the columns as the index to gain some intuition about how Dask leverages an index.

Create a pandas DataFrame with two columns of data, and a 2-partition Dask DataFrame from it.

```python
import dask.dataframe as dd
import pandas as pd

df = pd.DataFrame(
     {"col1": ["aa", "dd", "cc", "bb", "aa"], "col2": ["a1", "d", "c", "b", "a2"]}
)
ddf = dd.from_pandas(df, npartitions=2)
```

Print the DataFrame and see that it has one index column that was created by default by pandas and two columns with data.

```python
print(ddf.compute())

  col1 col2
0   aa   a1
1   dd    d
2   cc    c
3   bb    b
4   aa   a2
```

Take a look at the divisions of ddf.

```python
ddf.divisions

(0, 3, 4)
```

ddf has two divisions.  Let's create a print_partitions() helper method and print ddf to better illustrate the divisions.

```python
def print_partitions(ddf):
     for i in range(ddf.npartitions):
         print(ddf.partitions[i].compute())

print_partitions(ddf)

  col1 col2
0   aa   a1
1   dd    d
2   cc    c
  col1 col2
3   bb    b
4   aa   a2
```

The first partition contains values with indexes from 0 to 2.  The second division contains values with indexes 3 and above.

Create a new DataFrame with col1 as the index and print the result.

```python
ddf2 = ddf.set_index("col1", divisions=["aa", "cc", "dd"])

print(ddf2.compute())


col1  col2     
aa     a1
aa     a2
bb      b
cc      c
dd      d
```

Let's confirm that the divisions of ddf2 are set as we specified.

```python
ddf2.divisions

('aa', 'cc', 'dd')
```

Let's look at how the data is distributed in ddf2 by partition.

```python
print_partitions(ddf2)

col1  col2     
aa     a1
aa     a2
bb      b

col1  col2     
cc      c
dd      d
```

Notice that the set_index method sorted the DataFrame based on col1.  Take note of two important changes after set_index is run:

- The divisions of ddf2 are now set as specified
- The data is repartitioned, as per our division specifications. bb has moved to the first partition, and dd to the second, because that's what our divisions specified.
- Repeated values are all in the same partition. That is, both aa's are in the first partition.

Sorting a Dask DataFrame can be a slow computation, see [this blog post for more detail](https://coiled.io/blog/better-shuffling-in-dask-a-proof-of-concept/).

You will only want to run set_index when you can leverage it to make downstream queries faster.

Let's look at a big dataset and see how much faster some queries execute when an index is set.

## Reading DataFrames with index

Let's grab a few rows of data from a 662 million row DataFrame when filtering on the index.  Then, let's run the same query when the index isn't set to quantify the performance speedup offered by the index.

Read in the DataFrame using a 5 node Coiled cluster and verify that the timestamp is the index.

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_be9599a4-6c30-4743-aa64-3e10e4ddc6b6" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=be9599a4-6c30-4743-aa64-3e10e4ddc6b6&amp;signature=AAH58kHGX-5oDnf7-w1PgaJP8-8KRs0g9g&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=24978215-edb6-419d-8bbf-c721d11a9e55&amp;redirect_url=APefjpGygs2XfX8Ubi3nFEK7U7pBynqySrj2lvFeQUPuREabPSJaFVysiiXNUFqCtFfWTTd0e-R0WYJIbFKuzNpdhABo7C8PVVKdMUkRO5mgvDyZyGaV5zQ&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fdask-set-index-dataframe&amp;ts=1744161887615" style=" cta_dest_link="https://cloud.coiled.io/signup" title="Try Coiled"><div style="text-align: center;"><span style="font-family: Helvetica, Arial, sans-serif;"><span style="font-size: 16px;">Try </span><span style="font-size: 16px;">Coiled</span></span></div></a></span>
</span></div>

```python
import coiled
import dask

cluster = coiled.Cluster(name="demo-cluster", n_workers=5)

client = dask.distributed.Client(cluster)

ddf = dd.read_parquet(
     "s3://coiled-datasets/timeseries/20-years/parquet",
     storage_options={"anon": True, "use_ssl": True},
     engine="pyarrow",
)

ddf.head()
```

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5f9ab3551ed39bcd65_image-700x332.png" alt=">

Let's take a look at what Dask knows about the divisions of this DataFrame.

```python
ddf.known_divisions

True
```

You can print out all the divisions too if you'd like to inspect them.

```python
ddf.divisions

(Timestamp('2000-01-01 00:00:00'),
 Timestamp('2000-01-08 00:00:00'),
 Timestamp('2000-01-15 00:00:00'),
…
```

This DataFrame has 1,096 divisions.

Here's a query that'll grab three records from the DataFrame and takes 2 seconds to execute.

```python
//len(ddf.loc["2000-01-01 00:00:02":"2000-01-01 00:00:04"])
```

Read in the same dataset without an index, so we can see how much slower the same query runs on a regular column that's not the index.

```python
//ddf2 = dd.read_parquet(
    "s3://coiled-datasets/timeseries/20-years/parquet",
    storage_options={"anon": True, "use_ssl": True},
    engine="pyarrow",
    index=False,
)

len(
    ddf2.loc[
        (ddf2["timestamp"] >= "2000-01-01 00:00:02")
& (ddf2["timestamp"] <= "2000-01-01 00:00:04")
    ]
)
```

That query takes 115 seconds to execute because it had to load and scan all of the data looking for those values, instead of jumping straight to the correct partition.  The query without the index is 58 times slower in this example.

Filtering on an index value is clearly a lot quicker than filtering on non-index columns.

## How long does it take to set the index?

Let's look at a query to compute the number of unique names in the data when the id equals 1001.  We'll see if setting an index can improve the performance of the query.

Start by reading in the data and running a query to establish a baseline.

```python
ddf3 = dd.read_parquet(
     "s3://coiled-datasets/timeseries/20-years/parquet",
     storage_options={"anon": True, "use_ssl": True},
     engine="pyarrow",
     index=False,
)

ddf3.loc[ddf3["id"] == 1001].name.nunique().compute()
```

This query takes 75 seconds to execute.

dd3 has 1,095 partitions, which you can see by running ddf3.npartitions.

### Setting an index before running the query

Let's set an index and run the same query to see if that'll make the query run faster.

```python
dask_computed_divisions = ddf3.set_index("id").divisions
unique_divisions = list(dict.fromkeys(list(dask_computed_divisions)))
len(unique_divisions) # 149
ddf3u = ddf3.set_index("id", divisions=unique_divisions)
ddf3u.npartitions # 148
ddf3u.loc[[1001]].name.nunique().compute()
```

This query takes 130 seconds to run.  Setting the index before running the query is slower than simply running the query.  It's not surprising that this runs slower.  Setting the index requires a data sort, which is expensive.

This example just shows one way you can create the divisions that are used when the index is set.  You can create the divisions however you'd like.  Choosing the optimal divisions to make downstream queries faster is challenging.

It's not surprising that this particular index does not improve the overall query time: we're sorting *all* of the values, but we only needed to extract just one. It's faster to just scan the un-sorted data and find the single value we're interested in.

Let's look at some more examples to gather some intuition and then look and when setting an index is more likely to improve query times.

### Setting an index without specifying divisions

Let's look at setting an index without explicitly setting divisions and running the same count query.

```python
ddf3a = ddf3.set_index("id") # 134 seconds

ddf3a.loc[[1001]].name.nunique().compute() # 118 seconds
```

This takes a total of 252 seconds to compute and is