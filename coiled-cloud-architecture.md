---
title: Coiled Cloud Architecture
description: Over the last couple of years, Coiled has made a cloud-SaaS application that runs Dask for folks smoothly and securely in the cloud...
blogpost: true
date: 
author: 
---

# Coiled Cloud Architecture

Running Dask in the cloud is easy.  

Running Dask in the cloud securely is hard.

Running Dask for an enterprise in the cloud is *really* hard.

Over the last couple of years, Coiled has made a cloud-SaaS application that runs Dask for folks smoothly and securely in the cloud.  

We thought you would like to hear a bit about the choices we made and why.

## Design objectives

Coiled had three main objectives when making our design:

1. Easy for users
2. Trustworthy for IT
3. Easy for us to maintain

This led us down a path of a cloud SaaS application with:

- A centralized control plane to make our development faster
- It's ok for Coiled to see users' metadata
- Deployment of Dask clusters within user cloud accounts
- It's not ok for Coiled to see users' data
- As few opinions as possible, following a "best of breed" philosophy

We felt that this was the best choice to enable scalable Python for the most people in a way that we could execute on with a small team.  We're pretty happy with the result.

## Paths we didn't choose

There are lots of excellent paths here that make sense, but that we chose not to pursue.  For example:

1. Host computation on our servers, like Snowflake does

We felt that most customers wouldn't be comfortable giving a small company like us their data directly.  

2. Deploy entirely on customer accounts, like Cloudera does

We felt that this would make larger customers more comfortable with us, but would slow down our development cycle too much.

3. Build an end-to-end data science/engineering work environment, like Domino/Saturn Cloud/IBM/Databricks do

We had built these kinds of systems pre-Coiled and found that they're just really really hard to do well.  Data science environments were, in our opinion, too varied to give people a good experience.

These are all great choices, and target different kinds of users.  

## Coiled architecture

Coiled is a web application that manages cloud resources within user accounts on their behalf.  You give us highly restricted access to your cloud.  Members of your team then ask for Dask clusters, and we set up those Dask clusters for them on the fly.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/6520457a34a9fa5490a538f5_coiled-architecture.png" loading="lazy" alt="Architecture diagram showing how the user environment (eg your laptop) interacts with the Coiled Cloud application and your cloud computing environment (where resources are created from AWS or GCP).">

We've found that this approach provides a good balance of giving users the data security that they need while letting us iterate very quickly on the product and so provide a better user experience. It's a little nuanced though, let's dig in a bit.

## Admin users give Coiled permissions to manage cloud resources

First, let's go through how we coordinate cloud access safely to set up resources for users.

As a user you need to to give Coiled enough permissions to do things like: 

- Create and destroy instances
- Create and destroy network infrastructure
- Write and read logs into cloud services
- Write and read images into Docker registries

This is done once and then the Coiled web app has these permissions until you revoke them. Notably, you don't give any permission to read and write from sensitive data sources like cloud object stores, Snowflake, and so on. All Coiled can really do is run up your cloud bill.

## Users ask for Dask clusters

Then when users ask for a Dask cluster

```python
import coiled
cluster = coiled.Cluster(...)
```

Coiled looks up your credentials, creates the necessary infrastructure, creates trusted security tokens, and gives those tokens to either side so that the user can then connect directly from their system to that cluster.

```python
from dask.distributed import Client
client = Client(cluster)
```

Coiled sets up infrastructure, brokers a secure connection between the user and that infrastructure, and then gets out of the way as quickly as possible.

## Users send data access credentials to the cluster

The user then has access to a Dask cluster without Coiled acting in the middle. This is great. The user then sends data access credentials from their machine up to the Dask cluster. This allows the cluster to access sensitive data resources within the company without Coiled itself ever seeing the data or the credentials.

## Results

The result is that anyone on your team can quickly get access to a fully authenticated and secure Dask cluster that can access all of the data that they have access to in a minute or two.


Let's talk a bit about how we actually set up and deploy resources.

## Set up Dask workers in the cloud

To actually create the Dask scheduler and worker processes Coiled uses raw instances. This is in contrast to other options like using Kubernetes or container-as-a-service options like AWS Fargate.

Kubernetes is great. We love Kubernetes here at Coiled, but it also introduces complexity that we actually don't need to accomplish our goals. Also, removing the intermediate layer of Kubernetes allowed us to develop a product that was more flexible and raw-metal.

Instead, we use technologies like EC2 fleets so that we could easily adapt to our customers' many varied workloads with often very different hardware requirements. Once we have a bunch of VMs at our disposal we need to install just the right versions of Python packages on them. This can be tricky.

## Manage python libraries in the cloud

In order for Dask to work well, the versions of software libraries need to match exactly between where Coiled/Dask is invoked, and the workers that are run in the cloud. We do this in two ways (learn more in [our documentation](https://docs.coiled.io/user_guide/software/index.html)):

1. We enable users to ask for specific conda/pip packages, and build docker images to their specification.

```python
import coiled
coiled.create_software_environment(
    "prefect-flow",
    conda={"channels": ["conda-forge"],
           "dependencies": ["coiled-runtime", "prefect"]
)
```

This works well, but can get a bit finicky especially when the user's environment changes rapidly.  Adding a conda solve and docker build step into development cycles causes a lot of slowdown and pain.

2. We dynamically query the user's local environment and install those packages on-the-fly This is far more magical, but works pretty well:

```python
cluster = coiled.Cluster(package_sync=True)
```

## Handle Dask logs in the cloud

When something breaks users often want to go and look at logs. Where should these logs go?

As always, Coiled tries hard not to store anything from the user, and instead rely on cloud-native solutions.  This means that for AWS we store user logs in CloudWatch, and for Google Cloud we store user logs in Google Cloud Logging.  We provide easy methods to query these logs, but they are also easily accessible from however the user already handles logs within their organization.

As always, Coiled tries to be convenient, but not intrusive.

## 24x7 tracking of cluster status and Dask usage

Finally, Coiled also acts as a watchdog over your Dask processes.  If a cluster has been idle for too long we'll kill it.  If a user is launching GPU resources and not using them, we make folks aware of it.  Gathering usage information is key.

To accomplish this, Coiled installs a SchedulerPlugin in the Dask schedulers that tracks various usage metrics like idleness, which operations are running, etc. This information is *really* useful for debugging and cost optimization, but can also be somewhat invasive. We allow users to turn various bits of this data tracking on and off with configuration (see the [Coiled documentation on analytics](https://docs.coiled.io/user_guide/analytics.html)).

## Billing

Coiled is free for the first 10,000 CPU hours a month (although you still have to pay your cloud provider).  Beyond that we charge $0.05 per managed CPU hour, with rates that decline based on usage.

We've found that this is enough for individuals to operate entirely for free, and for folks at larger organizations to "try before they buy" pretty effectively. For companies who do spend thousands of dollars or more per month, we look comparable to services like Databricks or SageMaker. Especially when compared against the personnel cost of building and maintaining a system like Coiled, this very obviously becomes worth the cost.

## Results

Anyone with a cloud account can connect their accounts to Coiled in a couple of minutes, and can be scaling happily afterwards, knowing that they're secure. Our development teams have enough information to see what's going on to rapidly iterate on a better user experience, while users can rest assured that both their data and budgets are safe and secure. For a more details, see [Why Coiled?](https://docs.coiled.io/user_guide/why.html).

If you're interested, Coiled is easy to try out. Just run the following in any Python-enabled environment after you've created a Coiled account:

```python
pip install coiled
coiled login
coiled setup aws
ipython
```

```python
import coiled
cluster = coiled.Cluster(n_workers=100, package_sync=True)

from dask.distributed import Client
client = Client(cluster)

import dask.array as da
da.random.random((10000, 10000)).sum().compute()
```

‍

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id=""hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec"" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_be9599a4-6c30-4743-aa64-3e10e4ddc6b6" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=be9599a4-6c30-4743-aa64-3e10e4ddc6b6&amp;signature=AAH58kGJkZxFect0b57Wm-DFnCFCS5w6Rw&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=94db0b66-6730-46c8-85d4-6ef68df628fb&amp;redirect_url=APefjpG8ZSR5Le5WeXsW8oC4iVJFFDtKXgHdyG-8Bd54009iUpoWrs7sMgf-UalNKEYGdiFc656FR4HA-losHUjpryoxlnnhH304zQci1tJe3isiurm0ThY&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fcoiled-cloud-architecture&amp;ts=1744255826037" style="" cta_dest_link="https://cloud.coiled.io/signup" title="Try Coiled"><div style="text-align: center;"><span style="font-family: Helvetica, Arial, sans-serif;"><span style="font-size: 16px;">Try </span><span style="font-size: 16px;">Coiled</span></span></div></a></span>
</span>