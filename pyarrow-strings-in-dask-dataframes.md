---
title: PyArrow Strings in Dask DataFrames
description: Pandas 2.0 is released! PyArrow data types are a major part of this release, notably PyArrow strings, which are faster and more compact in memory than Python object strings, the historic solution...
blogpost: true
date: 
author: 
---

# PyArrow Strings in Dask DataFrames

pandas 2.0 has been released!  🎉

Improved PyArrow data type support is a major part of this release, notably for PyArrow strings, which are faster and more compact in memory than Python object strings, the historic solution. This change impacts pandas users everywhere, but especially impacts Dask DataFrame users, who often run at the capacity of their hardware.

PyArrow strings often use less memory, and so allow you to process more data on less hardware. In the example below we'll find that we can operate on the same data, faster, using a cluster of one third the size. This corresponds to about a 75% overall cost reduction.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/642c76ac9d63cb504c88a2f6_Arrow%20Strings%20vs%20Python%20Strings.png" loading="lazy" alt=">

## How to use PyArrow strings in Dask

```python
pip install pandas==2
```

```python
import dask
dask.config.set({"dataframe.convert-string": True})
```

Note, support isn't perfect yet. Most operations work fine, but some (any, all, …) still require upstream fixes. Thank you to pandas and PyArrow maintainers for their collaboration in this effort.

This option is available starting with Dask 2023.03.01 and it requires pandas 2.0 to be installed.

## Dask PyArrow Example

Let's consider the following example, where we load some public Uber/Lyft Parquet data onto a cluster running on the cloud.

```python
import coiled
from dask.distributed import wait
import dask.dataframe as dd
import dask

dask.config.set({"dataframe.convert-string": True})

cluster = coiled.Cluster(
	 n_workers=15,
   backend_options={"region": "us-east-2"},
 )
  
client = cluster.get_client()
 
df = dd.read_parquet(
  "s3://coiled-datasets/uber-lyft-tlc/",
  storage_options={"anon": True},
).persist()
wait(df)
```

This takes about 33 seconds to load. We can then do operations to find out, for example, how many rides are tipped, broken down by Uber and Lyft rides:

```python
df["tipped"] = df.tips != 0
df.groupby(df.hvfhs_license_num).tipped.mean().compute()
```

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/642c42dd6afda3ac621b9e89_dZlvhFBIjos4oiNtp97JyohHMhuZxvkMVnSQAFi13_fxaA0ZrZ-GHKvpLYKHla84SZKXFc_Xsqoj3T_N5jq44ORcoOB8ARBcugE4ncLjO2n7B-3DgjWQnVfsOp6TuEAl6M0TXrZ0vxdrNZ2ZTgY-2b0.png" loading="lazy" alt=">

If you'd like to play more with this dataset (it's fun!) there's a monthly free tutorial that you can sign up [here](https://coiled.io/tutorials).

## Turn off PyArrow Strings

We can run this exact same computation, but now without PyArrow strings:

```python
dask.config.set({"dataframe.convert-string": False})
```

Unfortunately, the first step to make this work is to ask for 3x the hardware. Otherwise the dataset doesn't fit into memory, and we're stuck repeatedly reading from storage:

```python
cluster.scale(45)
client.wait_for_workers(45)
```

Then everything works fine, and we get the same results as above, although operations like the groupby-aggregation above take twice as long (even with 3x the hardware).

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/642c42ddcdf853e5312cd8bd_ItmFOaS2WkqN5YcUXLvtPZlxPfw2lGtSh23K42XXuW-s-mvVV8XPr7rJzqWGk0s5Pabz4SqxZahkf8pxAo_PfJNScIIRKxprcaus6IeonbxTdH8hFd3eMxrnwlUwI_Xy50UaioVnLGuetc6tK-HelOU.png" loading="lazy" alt=">

## Learn, Try, Report

PyArrow strings are a significant win for the entire PyData community. They are also a work in progress. While we're excited to adopt them within our own benchmarking suites, we also know that early adopters will experience some rough edges. If you'd like to learn more, here are a few resources: 

- pandas: [What's new in pandas 2.0.0](https://pandas.pydata.org/docs/dev/whatsnew/v2.0.0.html)
- Uber/Lyft dataset: [Regular free tutorial playing with this dataset  ](https://www.coiled.io/tutorials)
- Dask + PyArrow: [Discussion issue where you can report experience, positive or negative](https://github.com/dask/dask/issues/10139)

If you have the time to use PyArrow strings with (or without) Dask and report your experience we'd love to hear about what's working well and what needs improvement.  

Thank you to everyone who contributed to this, but especially Matthew Roeschke (NVIDIA), Patrick Hoefler (Coiled), and Rick Zamora (NVIDIA) who worked to make sure all of the different libraries needed here came together.