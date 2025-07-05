---
title: Automate your ETL Jobs in the Cloud with Github Actions, S3 and Coiled
description: This post will demonstrate how running Github Actions on Coiled can be a useful way to schedule automated data-processing jobs...
blogpost: true
date: 
author: 
---

# Automate your ETL Jobs in the Cloud with Github Actions, S3 and Coiled

Github Actions let you launch automated jobs from your Github repository. Coiled lets you scale your Python code to the cloud. Combining the two gives you lightweight workflow orchestration for heavy ETL (extract-transform-load) jobs that can run in the cloud without any complicated infrastructure provisioning or DevOps.

[This Github Action](https://github.com/coiled/ci-with-coiled/blob/main/.github/workflows/quickstart.yaml) runs an ETL job on 15 GB of data and writes the results to an Amazon S3 bucket. Running this with a Github Action alone would not be possible as the default virtual machines on which Github Actions are run only have 7GB of memory. Running the action on a Coiled cluster lets you crunch the entire dataset. Since the Coiled cluster runs on AWS, the data can easily be written to an S3 bucket.

This post will demonstrate how running Github Actions on Coiled can be a useful way to schedule automated data-processing jobs. We'll start with some background on Github Actions and Coiled and then walk through an example script in detail.

## Github Actions, S3 and Coiled: Workflow Orchestration

Many data analysts have tasks that need to be run on a fixed schedule. For example, you might want to fetch new data every day at 8AM, preprocess it, and use it to train or update a machine learning model. Workflow orchestration tools allow you to schedule jobs like this without human intervention. Popular orchestration tools include Apache Airflow, Prefect, Argo and others.

Managing these workflow orchestration tools can be more work than you want to do. [Github Actions](https://github.com/features/actions) is a lightweight and cheap ([or even free)](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions) alternative that lets you run automated jobs based on predefined trigger events. Since many developers and data analysts already have their code in Github repositories, this can be an ideal way to run automated jobs without having to learn another tool or manage a full-fledged orchestration deployment.

## Github Actions, S3 and Coiled: Scale to the Cloud

Github Actions are run on virtual machines that are limited in the hardware resources they let you access. This makes sense because Github Actions are free (entirely for public repositories, up to a limit for private ones) and Github probably doesn't want you running massive workloads on their free service. By default, Github Actions are hosted on 'runners' with [7GB of RAM](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners).

So what do you do if you want to run a scheduled ETL job with Github Actions that requires more memory resources than the Github Action default?

If you want to use Github Actions for more compute-intensive or memory-bound tasks, you'll need to deploy the task to another virtual machine or cluster of machines. There are ways to do this yourself with Amazon ECS, Azure or Google Kubernetes. But deploying Github Actions to the cloud yourself is [complex](https://docs.github.com/en/actions/deployment/deploying-to-your-cloud-provider/deploying-to-amazon-elastic-container-service) and time-intensive. Github also offers an option to use [self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners) for Github Actions. These also require an extensive installation, setup and maintenance protocol.

Coiled makes it easy to deploy Python code to the cloud. Coiled handles all the infrastructure setup for you, including software environments, security and cost-management. Coiled Clusters are launched on-demand and can be stopped immediately once the computation is completed, so there will be no unexpected costs. Even better: Coiled currently provides a generous free tier, so you'll only be paying your own cloud account fees.

## Github Actions, S3 and Coiled: In Action

We've put together [a Github Action](https://github.com/coiled/ci-with-coiled/blob/main/.github/workflows/quickstart.yaml) that you can customize to run your own automated ETL jobs. Let's walk through the script in detail. If you're totally new to Github Actions, we recommend following [the official Github tutoria](https://docs.github.com/en/actions/quickstart)l first.

Let's break down the workflow that our Github Action performs step by step.

## Defining your Github Action

We'll set this ETL job to run on a schedule: it will be triggered at 12pm (noon) every day.

```python
on:
 schedule:
   # run at noon every day
   - cron: "0 12 * * *"
```

Github Actions uses [the cron format](https://www.netiq.com/documentation/cloud-manager-2-5/ncm-reference/data/bexyssf.html) to specify dates and time. You can also have your Github Actions triggered by other events, such as push events to specific repositories.

We'll then define our quickstart job:

```python
jobs:
 quickstart:
   runs-on: "ubuntu-latest"
   strategy:
     fail-fast: false
```

And define the steps this job will consist of:

```YAML
steps:
  # Needed to run on PR or branches
  - name: Checkout source
    uses: actions/checkout@v2
        
  - name: Set up local environment
    uses: conda-incubator/setup-miniconda@v2
    with:
      miniforge-variant: Mambaforge
      use-mamba: true
      environment-file: ci/environment-3.9.yaml
  
  - name: Run quickstart
    env:
      DASK_COILED__TOKEN: ${{ secrets.COILED_TOKEN }}
      AWS_ACCESS_KEY_ID: ${{ secrets.BLOG_AWS_ACCESS_KEY_ID }} 
      AWS_SECRET_ACCESS_KEY: ${{ secrets.BLOG_AWS_SECRET_ACCESS_KEY }}
    run: python scripts/quickstart.py
```

‍

The first and second steps are standard setup protocols. The third step is the one that will actually perform our ETL job: it will run the Python script quickstart.py located in the scripts directory.

## Authenticate with S3

Notice that we passed four environment variables to this script:

1. DASK_COILED__TOKEN contains the token your script will need to authenticate with Coiled and launch cloud computing resources. 
2. AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY contain your AWS credentials that let you authenticate with S3.

Secrets are pulled from your Github account settings. Follow [these steps](https://docs.github.com/en/actions/security-guides/encrypted-secrets) if you do not know how to store secrets in your Github repository.

## Running your ETL Job

Now let's take a closer look at the [quickstart.py script](https://github.com/coiled/ci-with-coiled/blob/main/scripts/quickstart.py) that our Github Action will run. 

After importing dependencies, the script starts by launching a Coiled cluster:

```python
GITHUB_RUN_ID = os.environ["GITHUB_RUN_ID"]


cluster = coiled.Cluster(
    name=f"github-actions-{GITHUB_RUN_ID}",
    n_workers=10,
    worker_memory="8Gib",
    account="dask-engineering",
)
```

This spins up a cluster of 10 virtual machines with 8GB of RAM each. Coiled automatically replicates the local Python environment in your cluster. You can [customize your cluster](https://docs.coiled.io/user_guide/api.html?highlight=coiled%20cluster#coiled.Cluster) to specify different instance types and hardware specs. Each cluster that is launched is given a unique name using the Github Run ID.

We then connect the Python client running the script (in this case the virtual machine that's running the Github Action) to our Coiled cluster:

```python
client = Client(cluster)
```

Now we're all set to load in our large dataset. We'll use Dask to do this. [Dask](https://dask.org/) is a parallel computing library for Python that makes it easy to scale your pandas and NumPy workloads to larger-than-memory datasets. Read [this blog](https://search.brave.com/search?q=speed+up+pandas&source=desktop) to learn how to speed up a pandas query 10x with Dask.

The code below loads the Parquet data and performs a groupby aggregation:

```python
ddf = dd.read_parquet(
    "s3://coiled-datasets/nyc-tlc/2019",
    columns=["passenger_count", "tip_amount"],
    storage_options={"anon": True},
).persist()

# perform groupby aggregation
result = ddf.groupby("passenger_count").tip_amount.mean()
```

## Write to S3

Chances are that once you have this result, you'd like to store it somewhere. Let's write the result to an Amazon S3 bucket. We'll write to a dedicated S3 bucket and create a unique directory for each Github Action run:

```python
# write result to s3
bucket_path = (
    f"s3://coiled-github-actions-blog/github-action-{GITHUB_RUN_ID}/quickstart.parquet"
)
result.to_parquet(
    bucket_path,
)
print(f"The result was successfully written to {bucket_path}")
```

Nobody likes an unexpected expensive cloud bill, so let's shut down our cluster with:

```python
client.close()
cluster.close()
```

All set! Once pushed to the repository, the Github Action will run this script at noon every day. You can of course adjust the cron schedule to run this at a different interval. After a few runs, your S3 bucket UI will look something like this, with a unique directory for each completed run:

‍

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/6372d950ee01afc01826807e_Coiled%20blog%20-%20github%20actions%201.png" loading="lazy" alt=">

‍

## Why Coiled?

While there are other ways to run your Github Actions on cloud-computing resources (or with self-hosted runners), these all require complicated setup processes and careful maintenance to implement security protocols and avoid unnecessary costs. Coiled takes care of all of this DevOps for you so you can focus on writing and running your Python code.

Coiled is also very scriptable and is not picky about where you are running your Python code. Unlike some of the other cloud-computing services, Coiled does not limit you to launching clusters from hosted notebooks. This means you can launch Coiled clusters from anywhere you can run Python code, including Github Actions.

At Coiled, we actually use this feature ourselves to [run benchmarking tests](https://github.com/coiled/coiled-runtime/blob/b36cd034ba9c57a42fe7be56b82fb8ac8d3f2e86/tests/benchmarks/test_parquet.py#L70) for the [coiled-runtime](https://github.com/coiled/coiled-runtime) metapackage on a ~200GB dataset.

## Github Actions, S3 and Coiled: Your turn

If you're ready to build your own Github Actions that leverage cloud-computing resources, follow these steps:

1. Fork the [template repository](https://github.com/coiled/github-actions-with-coiled) 
2. Create [a Coiled account](https://cloud.coiled.io/)
3. Edit the quickstart.py and quickstart.yaml files to suit your specific use case
4. Include your Coiled token and AWS S3 credentials in your Github settings
5. Launch your Github Action!

Note that spinning up Coiled clusters requires you to have an AWS or GCP account in which resources will be provisioned. Coiled currently provides 10,000 CPU hours free of charge. After your 10,000 CPU-hours, we charge $0.05 per CPU hour plus whatever you pay your cloud. We can't make your own cloud costs go away: you still have to pay AWS/GCP.

Let us know how you get on by [tweeting at us](https://twitter.com/CoiledHQ), we'd love to see what you build!