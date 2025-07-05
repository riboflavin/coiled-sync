---
title: Convert Large JSON to Parquet with Dask
description: You can use Coiled to convert large JSON data into a tabular DataFrame stored as Parquet in a cloud object store. Iterate locally first to build and test your pipeline, then transfer the same workflow to Coiled with only minimal code changes.
blogpost: true
date: 
author: 
---

# Convert Large JSON to Parquet with Dask

## tl;dr

You can use Coiled, the cloud-based Dask platform, to easily convert large JSON data into a tabular DataFrame stored as Parquet in a cloud object-store. Start off by iterating with Dask locally first to build and test your pipeline, then transfer the same workflow to Coiled with minimal code changes. We demonstrate a JSON to Parquet conversion for a 75GB dataset that runs without downloading the dataset to your local machine.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kENR6iqeQBGpq06VaVhiOfF7aEZ8A&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=355432ba-90df-47a5-acbf-52e7051abc87&amp;redirect_url=APefjpFD0wQ-sNyUDe5A4Y_TdAeD3NzloFsvwz54oolImvKTcBSfAO6N74iFBwFdgAycicSFx4IFL7yf4K1vCevtXWnK1JT8Lg726hxL8S2l36o5w-CxI80yztiUEtxxsWyV8HMPyDTa&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fconvert-large-json-to-parquet-with-dask&amp;ts=1744161550959" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>

* * *

## Why convert JSON to Parquet

Data scraped from the web in nested JSON format often needs to be converted into a tabular format for exploratory data analysis (EDA) and/or machine learning (ML). The Parquet file format is an optimal method for storing tabular data, allowing operations like column pruning and predicate pushdown filtering which [greatly increases the performance of your workflows](http://www.coiled.io/blog/parquet-file-column-pruning-predicate-pushdown) .

This post demonstrates a JSON to Parquet pipeline for a 75GB dataset, using Dask and Coiled to convert and store the data to a cloud object store. This pipeline bypasses the need for the dataset to be stored locally on your machine.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5b19ab6d694de63cb4_image-11-700x394.png" alt=">

Upon completing this notebook, you will be able to:

1. Build and test your ETL workflow locally first, using a single test file representing 1 hour of Github activity data
2. Scale that same workflow out to the cloud using Coiled to process the entire dataset.

*Spoiler alert -- y*ou'll be running the exact same code in both cases, just changing the place where the computations are run.

You can find a full-code example in [this notebook](https://github.com/coiled/coiled-resources/blob/main/blogs/convert-large-json-to-parquet-with-dask.ipynb). To run the notebook locally, build a conda environment with the environment.yml file located in the notebook repo.

## JSON to Parquet: Build your Pipeline Locally 

It's good practice to begin by building your pipeline locally first. The notebook linked above walks you through this process step-by-step. We'll just summarize the steps here.

1. **Extract a single file**

We'll be working with data from the [Github Archive project](https://www.gharchive.org/) for the year 2015. This dataset logs all public activity on Github and takes up ~75GB in uncompressed form. 

Begin by extracting a single file from the Github Archive. This represents 1 hour of data and takes up ~5MB of data. There's no need to work with any kind of parallel or cloud computing here, so you can iterate locally for now.

```python
!wget https://data.gharchive.org/2015-01-01-15.json.gz
```

*Only scale out to the cloud if and when necessary to avoid unnecessary costs and code complexity.*

2.  **Transform JSON data into DataFrame.**

Great, you've extracted the data from its source. Now you can proceed to transform it into tabular DataFrame format.** **There are several different schemas overlapping in the data, which means you can't simply cast this into a pandas or Dask DataFrame. Instead, you could filter out one subset, such as PushEvents, and work with that.

```python
records.filter(lambda record: record["type"] == "PushEvent").take(1)
```

You can apply the process function (defined in the notebook) to flatten the nested JSON data into tabular format, with each row now representing a single Github commit.

```python
records.filter(lambda record: record["type"] == "PushEvent").take(1)flattened = records.filter(lambda record: record["type"] == "PushEvent").map(process).flatten()
```

Then cast this data into a DataFrame using the .to_dataframe() method.

```python
df = flattened.to_dataframe()
```

3. **Load data to a local directory in Parquet file format**

You're now all set to write your DataFrame to a local directory as a .parquet file using the Dask DataFrame .to_parquet() method.

```python
df.to_parquet(
    "test.parq", 
    engine="pyarrow",
    compression="snappy"
)
```

## JSON to Parquet: Scaling out with Dask Clusters on Coiled

Great job building and testing out your workflow locally! Now let's build a workflow that will collect the data for a full year, process it, and save it to the cloud object storage.

1. **Spin up your Cluster**

We'll start by launching a Dask cluster in the cloud that can run your pipeline on the entire dataset. To run the code in this section, you'll need a free Coiled account by logging into [Coiled Cloud](http://cloud.coiled.io). You'll only need to provide your Github credentials to create an account.

You will then need to make a software environment with the correct libraries so that the workers in your cluster are able to execute our computations. 

```python
import coiled

# create Coiled software environment
coiled.create_software_environment(
    name="github-parquet",
    conda=["dask", "pyarrow", "s3fs", "ujson", "requests", "lz4", "fastparquet"],
)
```

*You can also create Coiled software environments using Docker images, environment.yml (conda), or requirements.txt (pip) files. For more information, check out the [Coiled Docs](https://docs.coiled.io/user_guide/software_environment_creation.html).*

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kENR6iqeQBGpq06VaVhiOfF7aEZ8A&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=355432ba-90df-47a5-acbf-52e7051abc87&amp;redirect_url=APefjpFD0wQ-sNyUDe5A4Y_TdAeD3NzloFsvwz54oolImvKTcBSfAO6N74iFBwFdgAycicSFx4IFL7yf4K1vCevtXWnK1JT8Lg726hxL8S2l36o5w-CxI80yztiUEtxxsWyV8HMPyDTa&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fconvert-large-json-to-parquet-with-dask&amp;ts=1744161550959" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>

Now, let's spin up your Coiled cluster, specifying the cluster name, the software environment it's running, and the number of Dask workers.

```python
# spin up a Coiled cluster
cluster = coiled.Cluster(
    name="github-parquet",
    software="coiled-examples/github-parquet", 
    n_workers=10, 
)
```

Finally, point Dask to run on computations on your Coiled cluster.

```python
# connect Dask to your Coiled cluster
from dask.distributed import Client
client = Client(cluster)
client
```

2. **Run your pipeline on the Github Archive dataset**

The moment we've all been waiting for! Your cluster is up and running, which means you're all set to run the JSON to Parquet pipeline you built above on the entire dataset. 

This requires 2 subtle changes to your code:
1. download *all* of the Github Archive files instead of just 1 test file
2. point df.to_parquet() to write to your s3 bucket instead of locally

Note that the code below uses a filenames list that contains all of the files for the year 2015 and the process function mentioned above. Refer to [the notebook](https://github.com/coiled/coiled-resources/blob/main/blogs/convert-large-json-to-parquet-with-dask.ipynb) for the definitions of these two objects.

```python
%%time
# read in json data
records = db.read_text(filenames).map(ujson.loads)

# filter out PushEvents
push = records.filter(lambda record: record["type"] == "PushEvent")

# process into tabular format, each row is a single commit
processed = push.map(process)

# flatten and cast to dataframe
df = processed.flatten().to_dataframe()

# write to parquet
df.to_parquet(
    's3://coiled-datasets/etl/test.parq',
    engine='pyarrow',
    compression='snappy'
)


CPU times: user 15.1 s, sys: 1.74 s, total: 16.8 s
Wall time: 19min 17s
```

Excellent, that works. But let's see if we can speed it up a little...

Let's scale our cluster up to boost performance. We'll use the cluster.scale() command to double the number of workers in our cluster. We'll also include a call to client.wait_for_workers() which will block activity until all of the workers are online. This way we can be sure that we're throwing all the muscle we have at our computation.

```python
# double n_workers
cluster.scale(20)

# this blocks activity until the specified number of workers have joined the cluster
client.wait_for_workers(20)
```

Let's now re-run the same ETL pipeline on our scaled cluster.

```python
%%time
# re-run etl pipeline
records = db.read_text(filenames).map(ujson.loads)
push = records.filter(lambda record: record["type"] == "PushEvent")
processed = push.map(process)
df = processed.flatten().to_dataframe()
df.to_parquet(
    's3://coiled-datasets/etl/test.parq',
    engine='pyarrow',
    compression='snappy'
)


CPU times: user 11.4 s, sys: 1.1 s, total: 12.5 s
Wall time: 9min 53s
```

We've cut the runtime in half, great job!

## Converting Large JSON to Parquet Summary

In this notebook, we showed how to convert JSON to Parquet by converting a raw JSON data into a flattened DataFrame and stored it in the efficient Parquet file format on a cloud object-store. We performed this workflow first on a single test file locally. We then scaled the same workflow out to run on the cloud using Dask clusters on Coiled in order to process the entire 75GB dataset.

Main takeaways:

- Coiled allows you to scale common ETL workflows to larger-than-memory datasets.
- Only scale to the cloud if and when you need to. Cloud computing comes with its own set of challenges and overhead. So be strategic about deciding if and when to import Coiled and spin up a cluster.
- Scale up your cluster for increased performance. We cut the runtime of the ETL function in half by scaling our cluster from 10 to 20 workers.

If you have any questions or suggestions for future material, feel free to drop us a line at support@coiled.io or in our [Coiled Community Slack channel](https://join.slack.com/t/coiled-users/shared_invite/zt-hx1fnr7k-In~Q8ui3XkQfvQon0yN5WQ). We'd love to hear from you!

 

## Try Coiled for Free

Thanks for reading. If you're interested in trying out Coiled, which provides hosted Dask clusters, docker-less managed software, and one-click deployments, you can do so for free today when you click below.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kENR6iqeQBGpq06VaVhiOfF7aEZ8A&amp;portal_id=9245528&amp;placement_guid=03d656