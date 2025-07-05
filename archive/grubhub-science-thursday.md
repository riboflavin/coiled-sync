---
title: Search at Grubhub and User Intent
description: Alex Egg, Senior Data Scientist at Grubhub, joins Matt Rocklin and Hugo Bowne-Anderson to talk and code about how Dask and distributed compute are used throughout the user intent classification pipeline at Grubhub!
blogpost: true
date: 
author: 
---

# Search at Grubhub and User Intent

Alex Egg, Senior Data Scientist at Grubhub, joins Matt Rocklin and Hugo Bowne-Anderson to talk and code about how Dask and distributed compute are used throughout the user intent classification pipeline at Grubhub!

**Context:** Search is the top-of-funnel at Grubhub. That means when a user interacts with the Grubhub search engine, they want to be able to service their request with high precision and recall. One way to do that is to understand the intent of the user. Do they have a favourite restaurant in mind or are they just browsing cuisines? Are they looking for an obscure dish? Depending on the intent of the user, Grubhub can optimize the backend query or apply UI treatments to help the user find just what they had in mind. In this session, we'll take a look at how to build an end-to-end search query intent classifier from data labeling to serving live requests in production.

We'll cover:

- Classic ETL pipelines with Dask Dataframes,
- Weak-supervision (labeling) w/ Snorkel & Dask (what a modern combo!),
- Language Modeling and deployment w/ Tensorflow.

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/_jqZzN2JRxQ?enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Search at Grubhub and User Intent: Dask, Snorkel, and TensorFlow!" data-gtm-yt-inspected-34277050_38="true" id="803931073" data-gtm-yt-inspected-12="true"></iframe>