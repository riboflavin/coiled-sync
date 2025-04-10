---
title: Dask on Azure
description: Dask is a flexible Python library for parallel and distributed computing. There are a number of ways you can create Dask clusters, each with their own benefits. In this article, we explore how Coiled provides a managed cloud infrastructure solution for Dask users.
blogpost: true
date: 
author: 
---

# Dask on Azure

‍[Dask](https://www.dask.org/) is a flexible Python library for parallel and distributed computing. There are a number of ways you can create Dask clusters, each with their own benefits. In this article, we explore how Coiled provides a managed cloud infrastructure solution for deploying Dask on Azure, addressing:

1. Deployment challenges
2. Software environment and credential management
3. Hardware flexibility
4. Historical tracking and observability
5. Cost management
6. Cost optimization

<figure style="padding-bottom:33.723653395784545%" class="w-richtext-align-center w-richtext-figure-type-video"><div><iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/eXP-YuERvi4?enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Six Coiled features for Dask users" data-gtm-yt-inspected-11="true" id="695636062" data-gtm-yt-inspected-34277050_38="true"></iframe></div></figure>

‍

## How to Deploy a Dask Cluster

There are a number of tools you can use to manage Dask clusters, including:

- [Dask Kubernetes](https://docs.coiled.io/user_guide/dask-deployment-comparisons.html#dask-kubernetes)
- [Dask Gateway](https://docs.coiled.io/user_guide/dask-deployment-comparisons.html#dask-gateway)
- [Dask Cloud Provider](https://docs.coiled.io/user_guide/dask-deployment-comparisons.html#dask-cloudprovider)

In principle, Coiled is just like these other Dask as a service products, in that you can also use Coiled to create a Dask cluster. While this might sound straightforward, there's significant nuance to deploying a Dask cluster effectively.

## Install Packages and Sync Credentials

On cluster start, Coiled examines your local machine, identifying pip and conda packages and any development packages you have locally. It packages these libraries and deploys them dynamically on cloud-based Linux machines in your cluster. This approach solves the [software environment matching problem](https://docs.coiled.io/user_guide/software/index.html) between your local and remote software environments. Additionally, Coiled securely transfers your local cloud credentials to the workers, ensuring they can access your cloud resources, like S3 buckets, safely and securely.

## Use GPUs, ARM, and Other Hardware

Coiled provides a range of options for configuring your Dask cluster, making it highly flexible. You can specify the number of virtual machines with n_workers and choose a region close to your remote data with the region option. Set the number of CPU cores with worker_cpu, [access GPUs](https://docs.coiled.io/user_guide/clusters/gpu.html) with worker_gpu, or opt for more efficient [ARM instances](https://docs.coiled.io/user_guide/clusters/arm.html) using arm=True.

```python
import coiled

cluster = coiled.Cluster(
    n_workers=100,
    worker_cpu=8,
    arm=True,
    region="us-east-1",
    spot_policy="spot_with_fallback",
)
```

By contrast, if you're using Kubernetes, you'd have to deploy a new Kubernetes cluster in this region with the node pool attached to a certain node type that has these kinds of machines in it. It can be kind of daunting.

Coiled's cluster configuration flexibility extends to AWS, GCP, and Azure allowing you to dynamically start machines of any type in any region, a powerful feature for hardware experimentation.

### Getting Started with Dask on Azure

Create Dask clusters on Azure by connecting [Coiled](https://docs.coiled.io/user_guide/setup/index.html) to your Azure account. Coiled starts virtual machines when you need them and turns them off when you're done. It's easy to set up Coiled to run from your Azure account:

```bash
$ pip install coiled
$ coiled login
$ coiled setup aws
```

Or you can skip the CLI and get started at [cloud.coiled.io](https://cloud.coiled.io/):

<figure style="padding-bottom:33.723653395784545%" class="w-richtext-align-center w-richtext-figure-type-video"><div><iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/d6XouzFP_AY?enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="How do I Set Up Coiled?" data-gtm-yt-inspected-11="true" data-gtm-yt-inspected-34277050_38="true" id="596097368"></iframe></div></figure>

## Observability Into Your Dask Cluster

Coiled provides [visibility and historical tracking](https://www.youtube.com/watch?v=4PiZf6UvCv0), offering insights into all activities for your team. You can monitor currently running and past clusters, with flags marking specific workload characteristics. For each cluster, you gain access to information about executed code, errors, and metrics. This visibility lets you optimize work across your team, even for members who are less familiar with the Dask codebase.

## Managing Cloud Costs

Effective cost management is crucial for organizations using cloud resources. Coiled offers team management features that enable you to limit the number of CPU cores running concurrently and on a monthly basis. You can also control team spending by monitoring usage. By default, all Coiled clusters will shutdown after 20 minutes of inactivity. These features are especially valuable for teams supporting users with varying skill levels, ensuring costs remain within budgetary constraints.

## Optimizing Cloud Costs

Coiled goes beyond cost management and helps optimize expenses in several ways, including minimizing network costs, using [cost-effective non-burstable instance types as the default](https://docs.coiled.io/blog/burstable-vs-nonburstable.html), and making it easy to use [efficient hardware like ARM instances](https://docs.coiled.io/blog/dask-graviton.html). There's nuance to offering this level of configurability and also creating a seamless user experience, though. One example of this is how Coiled helps you take advantage of spot virtual machines.

Though spot virtual machines are heavily discounted, most users are willing to pay more to avoid interruptions to their workflows as spot instances become unavailable. With Coiled, when you request spot instances, we look at your region and select the data center in that region with the best spot availability. Once we get the warning that these spot instances are being deprovisioned, we move your tasks onto new spot or on-demand instances. This ensures the continuity of your workloads without interruption. Learn more in [our blog post](/blog/save-money-with-spot).

## Conclusion

Coiled is a seamless managed Dask solution. It offers a smooth and secure cloud infrastructure experience for both individual users and organizations. Notably, you can use Coiled for free for the first 10,000 CPU hours per month, making it accessible to many users (you'll still be responsible for your Azure costs).

If you are currently using other Dask as a service products, like Dask Cloud Provider or Dask Kubernetes, Coiled offers cost savings and an improved user experience, making it worth considering to manage your Dask workflows in the cloud.

It's easy to [get started](https://docs.coiled.io/user_guide/setup/index.html) with Dask on Azure, GCP, or AWS.