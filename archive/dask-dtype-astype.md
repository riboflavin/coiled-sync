---
title: Reduce memory usage with Dask dtypes
description: This post gives an overview of DataFrame datatypes (dtypes), explains how to set dtypes when reading data, and shows how to change column types.
blogpost: true
date: 
author: 
---

# Reduce memory usage with Dask dtypes

Columns in Dask DataFrames are typed, which means they can only hold certain values (e.g. integer columns can't hold string values). This post gives an overview of DataFrame datatypes (dtypes), explains how to set dtypes when reading data, and shows how to change column types.

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_be9599a4-6c30-4743-aa64-3e10e4ddc6b6" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=be9599a4-6c30-4743-aa64-3e10e4ddc6b6&amp;signature=AAH58kE78Sz_wM1WPzLY--PD23oBJyQscg&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=23c0cf1e-623d-423b-b6c7-ba1b062f23a0&amp;redirect_url=APefjpFytgvxjTPeHFsOBPzO6q9PvqWmGLBowHgJ-RxZ8uS5sWNNAJ-Wah-aIvSu0bBFBCe45moYbl7ToU9y7SRVCk-i0y1YoK238lVXH5_v41HPgW4rxKI&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fdask-dtype-astype&amp;ts=1744161551844" style=" cta_dest_link="https://cloud.coiled.io/signup" title="Try Coiled"><div style="text-align: center;"><span style="font-family: Helvetica, Arial, sans-serif;"><span style="font-size: 16px;">Try </span><span style="font-size: 16px;">Coiled</span></span></div></a></span>
</span></div>

Using column types that require less memory can be a great way to speed up your workflows. Properly setting dtypes when reading files is sometimes needed for your code to run without error.

Here's what's covered in this post:

- Inspecting DataFrame column types (dtypes)
- Changing column types
- Inferring types when reading CSVs
- Manually specifying types
- Problems with type inference
- Using Parquet to avoid type inference

## Inspecting DataFrame types

Let's start by creating some DataFrames and viewing the dtypes.

Create a pandas DataFrame and print the dtypes.  All code snippets in this post are from [this notebook](https://github.com/coiled/coiled-resources/blob/main/dataframes/dask-dtypes.ipynb).

```python
import pandas as pd

df = pd.DataFrame({"nums":[1, 2, 3, 4], "letters":["a", "b", "c", "d"]})

print(df.dtypes)

nums       int64
letters    object
dtype: object
```

The nums column has the int64 type and the letters column has the object type.

Let's use this pandas DataFrame to create a Dask DataFrame and inspect the dtypes of the Dask DataFrame.

```python
import dask.dataframe as dd

ddf = dd.from_pandas(df, npartitions=2)

ddf.dtypes

nums       int64
letters    object
dtype: object
```

The Dask DataFrame has the same dtypes as the pandas DataFrame. 

## Changing column types

Change the nums column to int8. You can use Dask's astype function to cast an object to a different type.

```python
ddf['nums'] = ddf['nums'].astype('int8')

ddf.dtypes

nums        int8
letters    object
dtype: object
```

8 bit integers don't require as much memory as 64 bit integers, but can only hold values between -128 and 127, so this datatype is only suitable for columns that contain small numbers.

Selecting the best dtypes for your DataFrame is discussed in more detail later in this post.

## Inferring types

Let's read a 663 million row CSV time series dataset from a cloud filestore (S3) and print the dtypes.

We'll use the Coiled Dask platform to execute these queries on a cluster.

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_be9599a4-6c30-4743-aa64-3e10e4ddc6b6" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=be9599a4-6c30-4743-aa64-3e10e4ddc6b6&amp;signature=AAH58kE78Sz_wM1WPzLY--PD23oBJyQscg&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=23c0cf1e-623d-423b-b6c7-ba1b062f23a0&amp;redirect_url=APefjpFytgvxjTPeHFsOBPzO6q9PvqWmGLBowHgJ-RxZ8uS5sWNNAJ-Wah-aIvSu0bBFBCe45moYbl7ToU9y7SRVCk-i0y1YoK238lVXH5_v41HPgW4rxKI&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fdask-dtype-astype&amp;ts=1744161551844" style=" cta_dest_link="https://cloud.coiled.io/signup" title="Try Coiled"><div style="text-align: center;"><span style="font-family: Helvetica, Arial, sans-serif;"><span style="font-size: 16px;">Try </span><span style="font-size: 16px;">Coiled</span></span></div></a></span>
</span></div>

```python
import coiled
import dask

cluster = coiled.Cluster(name="demo-cluster", n_workers=5)
client = dask.distributed.Client(cluster)

ddf = dd.read_csv(
    "s3://coiled-datasets/timeseries/20-years/csv/*.part", 
    storage_options={"anon": True, 'use_ssl': True}
)

print(ddf.dtypes)

timestamp    object
id           int64
name         object
x            float64
y            float64
dtype: object
```

Dask infers the column types by taking a sample of the data. It doesn't analyze every row of data when inferring types because that'd be really slow on a large dataset.

Dask may incorrectly infer types because it only uses a sample.  If the inferred type is wrong then subsequent computations will error out. We'll discuss these errors and common work-arounds later in this post.

## Manually specifying types

This section demonstrates how manually specifying types can reduce memory usage.

Let's look at the memory usage of the DataFrame:

```python
ddf.memory_usage(deep=True).compute()

Index           140160
id          5298048000
name       41289103692
timestamp  50331456000
x           5298048000
y           5298048000
dtype: int64
```

The id column takes 5.3GB of memory and is typed as an int64. The id column contains values between 815 and 1,193. We don't need 64 bit integers to store values that are so small. Let's type id as a 16 bit integer and see how much memory that saves.

```python
ddf = dd.read_csv(
    "s3://coiled-datasets/timeseries/20-years/csv/*.part", 
    storage_options={"anon": True, 'use_ssl': True},
    dtype={
      "id": "int16"
    }
)

ddf.memory_usage(deep=True).compute()

Index           140160
id          1324512000
name       41289103692
timestamp  50331456000
x           5298048000
y           5298048000
dtype: int64
```

The id column only takes up 1.3 GB of memory when it's stored as a 16 bit integer.

It's not surprising that the memory requirements are 4 times lower because int16 is 4x smaller than int64 ;)

Manually specifying column dtypes can be tedious, especially for wide DataFrames, but it's less error prone than inferring types.

## Problems when inferring types

As mentioned earlier, Dask does not look at every value when inferring types. It only looks at a sample of the rows.

Suppose a column has the following values: 14, 11, 10, 22, and "hi". If Dask only samples 3 values (e.g. 11, 10, and 22), then it might incorrectly assume this is an int64 column.

Let's illustrate this issue with the New York City Parking Tickets dataset.

ddf = dd.read_csv(
    "s3:coiled-datasets/nyc-parking-tickets/csv/*.csv",
    storage_options={"anon": True, 'use_ssl': True})

len(ddf) # this works

ddf.head() # this errors out

---------------------------------------------------------------------------
ValueError                                Traceback (most recent call last)
/var/folders/d2/116lnkgd0l7f51xr7msb2jnh0000gn/T/ipykernel_9266/95726415.py in <module>
---> 1 ddf.head()

ValueError: Mismatched dtypes found in `pd.read_csv`/`pd.read_table`.

+-----------------------+---------+----------+
| Column                | Found   | Expected |
+-----------------------+---------+----------+
| House Number          | object  | float64  |
| Issuer Command        | object  | int64    |
| Issuer Squad          | object  | int64    |
| Time First Observed   | object  | float64  |
| Unregistered Vehicle? | float64 | int64    |
| Violation Description | object  | float64  |
| Violation Legal Code  | object  | float64  |
| Violation Location    | float64 | int64    |
| Violation Post Code   | object  | float64  |
+-----------------------+---------+----------+

The following columns also raised exceptions on conversion:

- House Number
  ValueError("could not convert string to float: '67-21'")
- Issuer Command
  ValueError("invalid literal for int() with base 10: 'T730'")
- Issuer Squad
  ValueError('cannot convert float NaN to integer')
- Time First Observed
  ValueError("could not convert string to float: '1134P'")
- Violation Description
  ValueError("could not convert string to float: 'BUS LANE VIOLATION'")
- Violation Legal Code
  ValueError("could not convert string to float: 'T'")
- Violation Post Code
  ValueError("could not convert string to float: 'H -'")

Usually this is due to Dask's dtype inference failing, and
*may* be fixed by specifying dtypes manually by adding:

dtype={'House Number': 'object',
       'Issuer Command': 'object',
       'Issuer Squad': 'object',
       'Time First Observed': 'object',
       'Unregistered Vehicle?': 'float64',
       'Violation Description': 'object',
       'Violation Legal Code': 'object',
       'Violation Location': 'float64',
       'Violation Post Code': 'object'}

to the call to `read_csv`/`read_table`.

House Number was inferred to be a float64, but it should actually be an object because it contains values like "67-21".

Let's look at the full set of dtypes in this DataFrame with print(ddf.dtypes).

//Summons Number                    int64
Plate ID                          object
Registration State                object
Plate Type                        object
Issue Date                        object
Violation Code                     int64
Vehicle Body Type                 object
Vehicle Make                      object
Issuing Agency                    object
Street Code1                       int64
Street Code2                       int64
Street Code3                       int64
Vehicle Expiration Date            int64
Violation Location                 int64
Violation Precinct                 int64
Issuer Precinct                    int64
Issuer Code                        int64
Issuer Command                     int64
Issuer Squad                       int64
Violation Time                    object
Time First Observed              float64
Violation County                  object
Violation In Front Of Or Opposite object
House Number                     float64
Street Name                       object
Intersecting Street               object
Date First Observed                int64
Law Section                        int64
Sub Division                      object
Violation Legal Code             float64
Days Parking In Effect            object
From Hours In Effect              object
To Hours In Effect                object
Vehicle Color                     object
Unregistered Vehicle?              int64
Vehicle Year                       int64
Meter Number                      object
Feet From Curb                     int64
Violation Post Code              float64
Violation Description            float64
No Standing or Stopping Violation float64
Hydrant Violation                float64
Double Parking Violation         float64
Latitude                         float64
Longitude                        float64
Community Board                  float64
Community Council                float64
Census Tract                     float64
BIN                              float64
BBL                              float64
NTA                              float64
dtype: object

Figuring out the right dtypes for each column is really annoying.

We can get past this initial error by specifying the dtypes, as recommended in the error message.

//ddf = dd.read_csv(
    "s3://coiled-datasets/nyc-parking-tickets/csv/*.csv",
    dtype={'House Number': 'object',
       'Issuer Command': 'object',
       'Issuer Squad': 'object',
       'Time First Observed': 'object',
       'Unregistered Vehicle?': 'float64',
       'Violation Description': 'object',
       'Violation Legal Code': 'object',
       'Violation Location': 'float64',
       'Violation Post Code': 'object'}
)

ddf.head()will work now but ddf.describe().compute() will error out with this message:

//---------------------------------------------------------------------------
ValueError                                Traceback (most recent call last)

ValueError: Mismatched dtypes found in `pd.read_csv`/`pd.read_table`.

+-------------------------+--------+----------+
| Column                  | Found  | Expected |
+-------------------------+--------+----------+
| Date First Observed     | object | int64    |
| Vehicle Expiration Date | object | int64    |
+-------------------------+--------+----------+

The following columns also raised exceptions on conversion:

- Date First Observed
  ValueError("invalid literal for int() with base 10: '01/05/0001 12:00:00 PM'")
- Vehicle Expiration Date
  ValueError("invalid literal for int() with base 10: '