---
title: Code Formatting Jupyter Notebooks with Black
description: This post explains how to use the plugin and why consistent code formatting is so important.
blogpost: true
date: 
author: 
---

# Code Formatting Jupyter Notebooks with Black

[Black](https://github.com/psf/black) is an amazing Python code formatting tool that automatically corrects your code.

The [jupyterlab_code_formatter](https://github.com/ryantam626/jupyterlab_code_formatter) extension makes it easy for you to run black in a Jupyter Notebook with the click of a button.

This post explains how to use the plugin and why consistent code formatting is so important.

## Installation

You can add the necessary libraries to your conda environment with this command:

```python
conda install -c conda-forge jupyterlab_code_formatter black isort
```

You can also add the three required dependencies to a YAML file if you're using a multi-environment conda workflow.

## Usage

Restart your Jupyter Lab session, and you'll now see this formatting button:

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b5a3a4f10fe585db590_image.png" alt=" loading="lazy">

Click the button to format all the cells in your notebook with Black's code formatting rules.

This notebook is running locally, but you can use Coiled to [run Jupyter Notebooks on the cloud](https://docs.coiled.io/user_guide/notebooks.html).

## Example code formatting

Let's take a look at a chunk of code that's not formatted and then see how it looks after it's been formatted.  Here's the unformatted code.

```python
ddf = dd.read_parquet(
   "s3://coiled-datasets/timeseries/20-years/parquet",
   storage_options={"anon": True, 'use_ssl': True}
)
```

This code uses inconsistent apostrophe symbols, and the last argument to read_parquet doesn't have a trailing comma.

Click the code formatting button, and this cell is automatically reformatted as follows:

```python
ddf = dd.read_parquet(
   "s3://coiled-datasets/timeseries/20-years/parquet",
   storage_options={"anon": True, "use_ssl": True},
)
```

You might feel like code formatting is minor, but it's actually really important.

## Why code formatting is important

Before automatic code formatting was popular, teams would create style guides and manually enforce coding rules.

This was tedious and sometimes contentious.  Some programmers are more passionate about their whitespace preferences than you might think!

Spending time creating style guides, commenting about formatting in pull requests, and arguing about double or single quotes is not a good use of developer time.  It's better to follow a community-accepted, fully automated solution.

## Other code formatting options

jupyterlab_code_formatter also supports YAPF and Autopep8 code formatting.  We recommend using Black because it's popular and used in several popular open-source libraries.

## Conclusion

JupyterLab has an extension that makes it easy to automatically format the Python code in your notebooks.

Automatic code formatters make it easy for teams to consistently style their code without extra work.