---
title: Snowflake and Dask: a Python Connector for Faster Data Transfer
description: This article discusses why and how to use both together, and dives into the challenges of bulk parallel reads and writes into data warehouses...
blogpost: true
date: 
author: 
---

# Snowflake and Dask: a Python Connector for Faster Data Transfer

**Snowflake** is a leading cloud data platform and SQL database. Many companies store their data in a Snowflake database.

**Dask** is a leading framework for scalable data science and machine learning in Python. 

This article discusses why and how to use both together, and dives into the challenges of bulk parallel reads and writes into data warehouses, which is necessary for smooth interoperability. It presents the new dask-snowflake connector as the best possible solution for fast, reliable data transfer between Snowflake and Python.

Here is a video walkthrough of what that looks like: 

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/pinFo1YBD-0?enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Dask Snowflake Integration | Tutorial on Snowflake and Dask | Richard Pelgrim" data-gtm-yt-inspected-34277050_38="true" id="239729757" data-gtm-yt-inspected-12="true"></iframe>

## Motivation

Data warehouses are great at SQL queries but less great at more advanced operations, like machine learning or the free-form analyses provided by general-purpose languages like Python.

That's ok. If you're working as a Python data scientist or data engineer, you're probably no stranger to extracting copies of data out of databases and using Python for complex data transformation and/or analytics.

It's common to use a database and/or data warehouse to filter and join different datasets, and then pass that result off to Python for more custom computations. We use each tool where it excels. This works well as long as it is easy to perform the handoff of large results.

However, as we work on larger datasets and our results grow beyond the scale of a single machine, passing results between a database and a general purpose computation system is challenging.

This article describes a few ways in which we can move data between Snowflake and Dask today to help explain why this is both hard and important to do well, namely:

1. Use pandas and the Snowflake connector

2. Break your query up into subqueries

3. Perform a bulk export to Apache Parquet

At the end, we'll preview the new functionality, which provides the best possible solution, and show an example of use.

## Three ways to pass data from Snowflake to Dask

There are different ways to accomplish this task, each with different benefits and drawbacks.  By looking at all three, we'll gain a better understanding of the problem, and what can go wrong.

### 1. Just use Pandas

First, if your data is small enough to fit in memory then you should just use pandas and Python.  There is no reason to use a distributed computing framework like Dask if you don't need it.

Snowflake publishes a Snowflake Python connector with pandas compatibility.  The following should work fine:

```python
$ pip install snowflake-connector-python

>>> import snowflake
>>> df = snowflake.fetch_pandas_all(...)
```

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/63867ce54255d6bde81e66f5_Snowflake%20Dask%20-%20pros%20and%20cons.jpg" loading="lazy" alt=">

### 2. Break into many subqueries

For larger datasets, we can also break one large table-scan query into many smaller queries and then run those in parallel to fill the partitions of a Dask dataframe.  

This is the approach implemented in the [dask.dataframe.read_sql_table](https://docs.dask.org/en/latest/dataframe-api.html#dask.dataframe.read_sql_table) function. This function does the following steps:

1. Query the length and schema of the table to determine the expected size, and determine an ideal number of partitions
2. Query for the minimum and maximum of a column on which to split
3. Submit many small queries that segment that column linearly between the minimum and maximum values.

For example, if we're reading from a time series then we might submit several queries which each pull off a single month of data.  These each lazily return a pandas dataframe.  We construct a Dask dataframe from these lazily generated subqueries. 

```python
import dask.dataframe as dd
df = dd.read_sql_table(
    'accounts', 
    'snowflake://user:pass@...warehouse=...role=...', 
    npartitions=10, 
    index_col='id'
)
```

This scales and works with any database that supports a SQLAlchemy Python connector (which Snowflake does).  However, there are a few problems:

- It only works for very simple queries, really just full table scans that can optionally select down columns and filter out rows.  For example, you couldn't run a big join query and then feed that result to Dask without first writing out to a temporary table
- It's kinda slow.  SQLAlchemy/ODBC connectors are rarely well optimized (see [Turbodbc](https://turbodbc.readthedocs.io/en/latest/) for a good counter-example)
- It's inconsistent.  Those subqueries aren't guaranteed to run at the same time, and so if other processes are writing to your database while you're reading you could get incorrect/inconsistent results.  In some applications, this doesn't matter.  In some, it matters a great deal

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/63867d2cc3e1fc7395544936_Snowflake%20Dask%20-%20pros%20and%20cons%202.jpg" loading="lazy" alt=">

For more information see this guide to [Getting Started with Dask and SQL](https://coiled.io/blog/getting-started-with-dask-and-sql/).

### 3. Bulk Export to Parquet

We can also perform a bulk export.  Both Snowflake and Dask (and really any distributed system these days) can read and write Parquet data on cloud object stores.  So we can perform a query with Snowflake and then write the output to Parquet, and then read in that data with Dask dataframe. 

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/63587d9db3c62b5abacab6a1_image1-1-700x632.png" alt=" loading="lazy">

```python
import dask.dataframe as dd
import snowflake
query = ""
    COPY INTO 's3://my_storage_location'
    from <Table name>
    file_format = (type = parquet)
    credentials = (aws_key_id='xxxx' aws_secret_key='xxxxx' aws_token='xxxxxx');
""
con = snowflake.connector.connect(user='XXXX', passwoard='XXXX', account='XXXX',)
con.curson().execute(query)
df = dd.read_parquet('s3://my_storage_location', ...)
```

This is great because now we can perform large complex queries and export those results at high speed to Dask in a fully consistent way.  All of the challenges of the previous approach are removed.  

However, this creates new challenges with data management.  We now have two copies of our data.  One in Snowflake and one in S3.  This is technically suboptimal in many ways (cost, storage, ...) but the major flaw here is that of data organization.  Inevitably these copies persist over time and get passed around the organization.  This results in many copies of old data in various states of disrepair.  We've lost all the benefits of centralized data warehousing, including cost efficiency, correct results, data security, governance, and more.  

Bulk exports provide performance at the cost of the organization.

*Although note, users interested in this option could also look into [Snowflake Stages](https://docs.snowflake.com/en/user-guide/data-unload-s3.html#unloading-data-into-an-external-stage).*

## New: Managed Bulk Transfers

Recently, Snowflake has added a capability for staged bulk transfers to external computation systems like Dask.  It combines the raw performance and support for complex queries of bulk exports, with the central management of directly reading SQL queries from the database.

Using this, we're able to provide an excellent Snowflake <-> Dask data movement tool. 

## Understanding Snowflake's Parallel Data Exports

Now Snowflake can take the result of any query, and stage that result for external access.

After running the query, Snowflake gives us back a list of pointers to chunks of data that we can read. We can then send pointers to our workers and they can directly pull out those chunks of data. This is very similar to the bulk export option described above, except that now rather than us being responsible for that new Parquet-on-S3 dataset, Snowflake manages it. This gives us all of the efficiency of a bulk export, while still letting Snowflake maintain control of the resulting artifact. Snowflake can clean up that artifact for us automatically so that we're always going back to the database as the single source of truth as we should.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kGVsenYWnxt79OE29ecvXRxtrtD1A&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=1d056701-b279-4793-896d-1f8a9b6b075e&amp;redirect_url=APefjpE5PuU7PwFEo5lkakHb68L6Z_7EE9pBVELtbMIDyYcbq9vB82ETK1qDeMPNM3UpmoWtkDLEyZ3TBDm6nYAoU31wJP_Wfbx_eBLT1VcAlkMUUF6pExTETSTf-UmVTA7tEh4PC2Gt&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fsnowflake-and-dask&amp;ts=1744162586134" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>

## Snowflake + Dask Integration

Given this new capability, we then wrapped Dask around it, resulting in the following experience:

```python
import dask_snowflake
import snowflake
with snowflake.connector.connect(...) as conn:
   ddf = dask_snowflake.from_snowflake(
      query=""
      SELECT * FROM TableA JOIN TableB ON ...
      "",
      conn=conn,
   )
```

This is a smooth and simple approach that combines the best of all previous methods.  It is easy to use, performs well, and maturely handles any query.

For more details on how to use distributed XGBoost training with Dask, see [this GitHub repo](https://github.com/coiled/dask-xgboost-nyctaxi).

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/pinFo1YBD-0?enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Dask Snowflake Integration | Tutorial on Snowflake and Dask | Richard Pelgrim" data-gtm-yt-inspected-34277050_38="true" id="445002999" data-gtm-yt-inspected-12="true"></iframe>

## Snowflake/Dask Fine Print

There are a few things to keep in mind:

1. We recommend using Dask on the same cloud and region as your Snowflake deployment for optimal speed and to avoid data ingress/egress charges.
2. The performance of your Dask queries will be impacted by the size and concurrency settings of your Snowflake Warehouse.
3. Your driving Python session (where you're importing dask-snowflake) needs to be properly authenticated with Snowflake.  Your remote workers do not.  Dask-snowflake doesn't require any additional considerations for authentication beyond what you do normally.  However, because authentication information is included in the messages sent to each worker we encourage you to ensure that you use secure network connections.
4. The Snowflake Python connector has some known issues on Apple M1 machines. See [this Github issue](https://github.com/snowflakedb/snowflake-connector-python/issues/986) for more details.

## Give it a spin!

[Download the dask-snowflake Jupyter Notebook to get started.](https://github.com/coiled/coiled-resources/blob/main/blogs/snowflake-and-dask.ipynb) If you run into any issues or have thoughts about further improvements to the package, come find us on [the Dask Discourse](https://dask.discourse.group/). 

*The technical work described in this post was done by various engineers at Snowflake, and James Bourbeau and Florian Jetter from Coiled.* *Watch James Bourbeau's talk at PyData Global 2021 together with Mark Keller and Miles Adkinds from Snowflake for a detailed walkthrough.*

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/WJ5yamxarfM?enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Snowflake &amp; Dask - Miles Adkins, James Bourbeau, Mark Keller | PyData Global 2021" data-gtm-yt-inspected-34277050_38="true" id="69709344" data-gtm-yt-inspected-12="true"></iframe>