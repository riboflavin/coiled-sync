---
title: Scikit-learn + Joblib: Scale your Machine Learning Models for Faster Training
description: You can train a sklearn models in parallel using the sklearn joblib interface. This allows sklearn to take full advantage of the multiple cores in your machine and speed up training.
blogpost: true
date: 
author: 
---

# Scikit-learn + Joblib: Scale your Machine Learning Models for Faster Training

You can train scikit-learn models in parallel using the [scikit-learn joblib interface](https://scikit-learn.org/stable/computing/parallelism.html#higher-level-parallelism-with-joblib). This allows scikit-learn to take full advantage of the multiple cores in your machine (or, spoiler alert, on your cluster) and speed up training.

Using the Dask joblib backend, you can maximize parallelism by scaling your scikit-learn model training out to a remote cluster.

This works for all scikit-learn models that expose the *n_jobs* keyword, like random forests, linear regressions, and other machine learning algorithms. You can also use the joblib library for distributed hyperparameter tuning with grid search cross-validation.

## Scikit-learn Models vs Data Size

When training machine learning models, you can run into 2 types of scalability issues: your model size may get too large, or your data size may start to cause issues (or even worse: both!). You can typically recognize a scalability problem related to your model size by long training times: the computation will complete (eventually), but it's becoming a bottleneck in your data processing pipeline. This means you're in the top-left corner of the diagram below:

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b61761c5f00ce2a8360_unnamed.png" alt=">

Using the scikit-learn joblib integration, you can address this problem by processing your scikit-learn models in parallel, distributed over the multiple cores in your machine.

Note that this is only a good solution if your training data and test data fit in memory comfortably. If this is not the case (and you're in the "I give up!" quadrant of large models and large datasets), see the "Dataset Too Large?" section below to learn how Dask-ML or XGBoost-on-Dask can help you out.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b61761c5f1f022a835f_unnamed-2-1.png" alt="sklearn joblib">

## Sklearn + Joblib: Run Random Forest in Parallel Locally

Let's see this in action with an scikit-learn algorithm that is embarrassingly parallel: a random forest model. A random forest model fits a number of decision tree classifiers on sub-samples of the training data , then combines the results from the various decision trees to make a prediction. Because each decision tree is an independent process, this model can easily be trained in parallel.

First, we'll create a new virtual environment for this work. (If you're following along, you can also use pip or any other package manager, or your own existing environment.)

```bash
conda create -n sklearn-example -c conda-forge python=3.9 scikit-learn joblib ipython
conda activate sklearn-example
```

We'll import the necessary libraries and create a synthetic classification dataset.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import make_classification
X, y = make_classification(n_samples=1000, n_features=4,
                          n_informative=2, n_redundant=0,
                          random_state=0, shuffle=False)
```

Then we'll instantiate our Random Forest classifier

```python
clf = RandomForestClassifier(max_depth=2, random_state=0)
```

We can then use a joblib context manager to train this classifier in parallel, specifying the number of cores we want to use with the n_jobs argument.

```python
import joblib
with joblib.parallel_backend(backend='loky', n_jobs=4):
    clf.fit(X, y)
```

To save users from having to use context managers every time they want to train a model in parallel, scikit-learn's developers exposed the n_jobs argument as part of the instantiating call:

```python
clf = RandomForestClassifier(max_depth=2, random_state=0, n_jobs=-1)
```

The n_jobs keyword communicates to the joblib backend, so you can directly call clf.fit(X, y) without wrapping it in a context manager. **This is the recommended approach for using joblib to train sklearn models in parallel locally.**

Running this locally with n_jobs = -1 on a MacBook Pro with 8 cores and 16GB of RAM takes just under 3 minutes.

```python
%%time
clf.fit(X,y)
CPU times: user 13min 21s, sys: 17.8 s, total: 13min 38s
Wall time: 2min 6s
```

## Sklearn + Joblib: Run RF on the Cluster

The context manager syntax can still come in handy when your model size increases beyond the capabilities of your local machine, or if you want to train your model even faster. In those situations, you could consider scaling out to remote clusters on the cloud. This can be especially useful if you're running heavy grid search cross-validation, or other forms of hyperparameter tuning.

You can use the Dask joblib backend to delegate the distributed training of your model to a Dask cluster of virtual machines in the cloud.

We first need to install some additional packages:

```bash
conda install -c conda-forge coiled dask xgboost dask-ml
```

Next, we'll launch a Dask cluster of 20 machines in the cloud.

When you create the cluster, Coiled will automatically replicate your local *sklearn-example* environment in your cluster (see [Package Sync](https://docs.coiled.io/user_guide/package_sync.html)).

```python
import coiled
cluster = coiled.Cluster(
    name="sklearn-cluster",
    n_workers=20,
)
```

We'll then connect Dask to our remote cluster. This will ensure that all future Dask computations are routed to the remote cluster instead of to our local cores.

```python
# point Dask to remote cluster
client = cluster.get_client()
```

Now use the joblib context manager to specify the Dask backend. This will delegate all model training to the workers in your remote cluster:

```python
%%time
with joblib.parallel_backend("dask"):
    clf.fit(X, y)
```

Model training is now occurring in parallel on your remote Dask cluster.

This runs in about a minute and a half on my machine—that's a 2x speed-up. You could make this run even faster by adding more workers to your cluster.

```python
CPU times: user 1.93 s, sys: 601 ms, total: 2.53 s
Wall time: 1min 1s
```

You can also use your Dask cluster to run an extensive hyperparameter grid search that would be too heavy to run on your local machine:

```python
from sklearn.model_selection import GridSearchCV

# Create a parameter grid
param_grid = {
    'bootstrap': [True],
    'max_depth': [80, 90, 100, 110],
    'max_features': [2, 3],
    'min_samples_leaf': [3, 4, 5],
    'min_samples_split': [8, 10, 12],
    'n_estimators': [100, 200, 300, 1000]
}

# Instantiate the grid search model
grid_search = GridSearchCV(
    estimator=clf, 
    param_grid=param_grid, 
    cv=5, 
    n_jobs=-1
)
```

Before we execute this compute-heavy Grid Search, let's just scale up our cluster to 100 workers to speed up training and ensure we don't run into memory overloads:

```python
cluster.scale(100)
client.wait_for_workers(100)
```

Now let's execute the grid search in parallel:

```python
%%time
with joblib.parallel_backend("dask"):
    grid_search.fit(X, y)
```

*If this is your first time working in a distributed computing system or with remote clusters, you may want to consider reading [The Beginner's Guide to Distributed Computing](https://towardsdatascience.com/the-beginners-guide-to-distributed-computing-6d6833796318).*

## Dataset too large?

Is your dataset becoming too large, for example because you have acquired new data? In that case, you may be experiencing two scalability issues at once: both your model and your dataset are becoming too large. You typically notice an issue with your dataset size by pandas throwing a MemoryError when trying to run a computation over the entire dataset.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b61761c5f37b22a8361_unnamed-3.png" alt="sklearn joblib">

Dask exists precisely to solve this problem. Using Dask, you can scale your existing Python data processing workflows to larger-than-memory datasets. You can use [Dask DataFrames](https://coiled.io/blog/speed-up-pandas-query-10x-with-dask/) or [Dask Arrays](https://docs.dask.org/en/stable/array.html) to process data that is too large for pandas or NumPy to handle. If you're new to Dask, check out this [Introduction to Dask](https://coiled.io/blog/what-is-dask/).

Dask also scales machine learning for larger-than-memory datasets. Use [dask-ml](https://ml.dask.org/) or the distributed [Dask backend to XGBoost](https://xgboost.readthedocs.io/en/stable/tutorials/dask.html) to train machine learning models on data that doesn't fit into memory. Dask-ML is a library that contains parallel implementation of many scikit-learn models.

The code snippet below demonstrates training a Logistic Regression model on larger-than-memory data in parallel.

```python
from dask_ml.linear_model import LogisticRegression
from dask_ml.datasets import make_classification

X, y = make_classification(chunks=50)
lr = LogisticRegression()
lr.fit(X, y)
```

Dask also integrates natively with XGBoost to train XGBoost models in parallel:

```python
import xgboost as xgb

# create DMatrix
dtrain = xgb.dask.DaskDMatrix(client, X, y)

# train model
output = xgb.dask.train(
   client, params, dtrain, num_boost_round=4,
   evals=[(dtrain, 'train')]
)
```

## Scikit-learn + Joblib Summary

We've seen that:

- You can use the scikit-learn's joblib integration to distribute certain scikit-learn tasks over all the cores in your machine for a faster runtime.
- You can connect joblib to the Dask backend to scale out to a remote cluster for even faster processing times.
- You can use XGBoost-on-Dask and/or dask-ml for distributed machine learning training on datasets that don't fit into local memory.

If you'd like to scale your Dask work to the cloud, Coiled provides quick and on-demand Dask clusters along with tools to manage environments, teams, and costs. Click below to learn more!

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kE0CUbWUnU55P0mJQ1PTod24uvShg&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=6dbb9dce-2e61-4c35-879e-db5696eca821&amp;redirect_url=APefjpHL1qpF2BUC7Dw0B3pq5l69kWh5HpWqJAqcQMuhESIXre70BAmKPKYtjfegEfJLWRWooITahn4PyIH-93gPG-glIzEkYmlp2o5sBD3ES9sHtMHXibIQiVd3ikxdf18Hx5EG4T6x&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fsklearn-joblib-dask&amp;ts=1744162586517" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>