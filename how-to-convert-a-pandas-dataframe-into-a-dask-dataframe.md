---
title: How to Convert a pandas Dataframe into a Dask Dataframe
description: Pandas is a very powerful Python library for manipulating and analyzing structured data, but it has some limitations. Dask can help.
blogpost: true
date: 
author: 
---

# How to Convert a pandas Dataframe into a Dask Dataframe

In this post, we will cover:

- How (and when) to convert a pandas DataFrame into a Dask DataFrame;
- Demonstrating 2x (or more!) speedup with an example;
- Discuss some best practices.

pandas is a very powerful Python library for manipulating and analyzing structured data, but it has some limitations. pandas does not leverage parallel computing capabilities of modern systems and throws a MemoryError when you try to read larger-than-memory data. Dask can help solve these limitations. [Dask](https://coiled.io/blog/what-is-dask/) is a library for parallel and distributed computing in Python. It provides a collection called Dask DataFrame that helps parallelize the pandas API.

This data is stored in a public bucket and you can use Coiled for free if you'd like to run these computations yourself.

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kHlCoVBVtr5Qv9tLUvZhGH2uR1dnw&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=8c8fe92c-80bd-40eb-8287-6c36013d9a49&amp;redirect_url=APefjpFPKt8nisQ_fwFeFhYDMuhFn8Q_lygFOyzxohu5JDAavjTdKwvGPSSDgqHYcMfDxUUVUuCyY3JBlwjaW6SCLYN1XfPCnIxgofa81102CsVzn8sO988xa_cVlvnNnKcXPS63pDbB&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fhow-to-convert-a-pandas-dataframe-into-a-dask-dataframe&amp;ts=1744161887574" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span></div>

You can follow along with the code in [this notebook](https://github.com/coiled/coiled-resources/blob/master/pandas-to-dask-dataframe/pandas-to-dask-dataframe.ipynb)!

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/l9c08OAT7jY?enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="How to Convert a pandas Dataframe into a Dask Dataframe | Pavithra Eswaramoorthy" data-gtm-yt-inspected-34277050_38="true" id="655033383" data-gtm-yt-inspected-12="true"></iframe>

## Analyzing library data with pandas 

Let's start with a sample pandas workflow of reading data into a pandas DataFrame and doing a groupby operation. We read a subset of the [Seattle Public Library checkouts dataset](https://www.kaggle.com/city-of-seattle/seattle-checkouts-by-title?select=checkouts-by-title.csv) (~4GB) and find the total number of items checked out based on the 'Usage Class' (i.e., physical vs. digital).

```python
df = pd.read_csv("checkouts-subset.csv") #

df.groupby("UsageClass").Checkouts.sum() # ~1.2 seconds
```

The computation takes about 1.2 seconds on my computer. Can we speed this up using Dask DataFrame? 

## Parallelizing using Dask DataFrame

To leverage single-machine parallelism for this analysis, we can convert the pandas DataFrame into a Dask DataFrame. A Dask DataFrame consists of multiple pandas DataFrames split across an index as shown in the following diagram:

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/638657ff104d4050037d09ec_DataFrame%20Image%20-%20Blue%202%403x.png)

We can use Dask's from_pandas function for this conversion. This function splits the in-memory pandas DataFrame into multiple sections and creates a Dask DataFrame. We can then operate on the Dask DataFrame in parallel using its pandas-like interface. Note that we need to specify the number of partitions or the size of each chunk while converting.

```python
ddf = dd.from_pandas(df, npartitions=10)
```

This conversion might take a minute, but it's a one-time cost. We can now perform the same computation as earlier:

```python
ddf.groupby("UsageClass").Checkouts.sum().compute() # ~679 ms
 seconds
```

This took ~680 milliseconds on my machine. That's more than twice as fast as pandas!

It's important to note here that Dask DataFrame provides parallelism, but it may not always be faster than pandas. For example, operations that require sorting take more time because data needs to be moved between different sections. We suggest sticking with pandas if your dataset fits in memory.

If you work with sizable data, it is likely stored on a remote system across multiple files, and not on your local machine. Dask allows you to read this data in parallel, which can be much faster than reading with pandas.

You can read data into a Dask DataFrame directly using Dask's read_csv function:

```python
import dask.dataframe as dd
ddf = dd.read_csv("s3://coiled-datasets/checkouts-subset.csv")
```

Both pandas and Dask also support several file-formats, including Parquet and HDF5, data formats optimized for scalable computing. You can check out the list of formats in the [pandas documentation](https://pandas.pydata.org/docs/user_guide/io.html#io).

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kHlCoVBVtr5Qv9tLUvZhGH2uR1dnw&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=8c8fe92c-80bd-40eb-8287-6c36013d9a49&amp;redirect_url=APefjpFPKt8nisQ_fwFeFhYDMuhFn8Q_lygFOyzxohu5JDAavjTdKwvGPSSDgqHYcMfDxUUVUuCyY3JBlwjaW6SCLYN1XfPCnIxgofa81102CsVzn8sO988xa_cVlvnNnKcXPS63pDbB&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fhow-to-convert-a-pandas-dataframe-into-a-dask-dataframe&amp;ts=1744161887574" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span></div>

## Using Dask  Distributed Scheduler

Dask DataFrame uses[ the single-machine threaded scheduler](https://docs.dask.org/en/latest/scheduling.html#local-threads) by default, which works well for local workflows. Dask also provides a [distributed scheduler](https://docs.dask.org/en/latest/scheduling.html#dask-distributed-local) with a lot more features and optimizations. It can be useful even for local development. For instance, you can take advantage of Dask's diagnostic dashboards. To use the distributed scheduler, use:

```python
from dask.distributed import Client
client = Client()
```

Then, perform your Dask computations as usual:

```python
import dask.dataframe as dd
ddf = dd.read_csv("https://coiled-datasets.s3.us-east-2.amazonaws.com/seattle-library-checkouts/checkouts-subset.csv")
ddf.groupby("UsageClass").Checkouts.sum().compute()
```

Dask is also capable of distributed computing in the cloud and we're building a service, Coiled, that allows you to deploy Dask clusters on the cloud effortlessly. Coiled offers a free tier with up to 100 CPU cores and 1000 CPU hours of computing time per month. Try it out today, and let us know how it goes!

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kHlCoVBVtr5Qv9tLUvZhGH2uR1dnw&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=8c8fe92c-80bd-40eb-8287-6c36013d9a49&amp;redirect_url=APefjpFPKt8nisQ_fwFeFhYDMuhFn8Q_lygFOyzxohu5JDAavjTdKwvGPSSDgqHYcMfDxUUVUuCyY3JBlwjaW6SCLYN1XfPCnIxgofa81102CsVzn8sO988xa_cVlvnNnKcXPS63pDbB&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fhow-to-convert-a-pandas-dataframe-into-a-dask-dataframe&amp;ts=1744161887574" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span></div>