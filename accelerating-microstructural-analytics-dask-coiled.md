---
title: Accelerating Microstructural Analytics with Dask and Coiled
description: In this article, we discuss an interesting use case of Dask and Coiled: Accelerating Volumetric X-ray Microstructural Analytics using distributed and high-performance computing.
blogpost: true
date: 
author: 
---

# Accelerating Microstructural Analytics with Dask and Coiled

In this article, we will discuss an interesting use case of Dask and Coiled: **Accelerating Volumetric X-ray Microstructural Analytics using distributed and high-performance computing**. This blog post is inspired by the [article published in Kitware Blog](https://blog.kitware.com/accelerating-volumetric-x-ray-microstructural-analytics-with-dask-itk-from-supercomputers-to-the-cloud/) and corresponding [research](https://ieeexplore.ieee.org/document/9307950).

We will explore:

- Research problem - analyzing large scale volumetric microstructural data
- Environment and methods used for the research
- Role of Dask and Coiled

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b569ab355682c9bcd40_visualization-700x368.gif" alt="">
<figcaption><em>Source: </em><a href="https://blog.kitware.com/accelerating-volumetric-x-ray-microstructural-analytics-with-dask-itk-from-supercomputers-to-the-cloud/" target="_blank"><em>blog.kitware.com</em></a></figcaption>

Do you have any exciting Dask or Coiled showcases to share? Tweet them to us [@CoiledHQ](https://twitter.com/coiledhq?lang=en), we would love to hear about it!

## Dask for Parallel and Distributed Computing

Dask is a library for parallel and distributed computing in Python. It helps scale machine learning workflows and provides a powerful framework to build distributed applications. Its familiar API, flexibility, and seamless integration with the Scientific Python Ecosystem make it a popular choice among data scientists and researchers.

Dask is used by academics and industry professionals across a variety of domains from life sciences and geophysical sciences to finance and retail corporations. It is also used behind-the-scenes by other software libraries like RAPIDS, Apache Airflow, Prefect, and many more. Learn more about Dask in [What is Dask?](https://coiled.io/blog/what-is-dask/) and [Who uses Dask?](https://coiled.io/who-uses-dask/)

In this showcase, we see Dask being used on a personal laptop, a supercomputer, and on the cloud with Coiled for accelerating the analysis of microstructural data in materials sciences.

## Microstructural Analytics for Material Science

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b569ab355ac399bcd43_pipeline-1024x352.png" alt="Fiber segmentation pipeline: (a) raw data with a virtual cut, (b) chunck after Hessian-based tubeness, (c) rough fiber segmentation.">
<figcaption>Fibre Segmentation pipeline.<br>Source: <a href="https://github.com/dani-lbnl/SC20_pyHPC/blob/master/SC20_pyHPC_dask4volumes.pdf" target="_blank">Article presented at Super Computing 2020</a></figcaption>

X-Ray Imaging is widely used to study new materials and to measure their properties. Advances in X-Ray technology at [synchrotron light sources](https://en.wikipedia.org/wiki/Synchrotron_light_source) greatly increase the size of datasets, which has led to efforts in automating analysis. So far, the efforts have been promising, but three-dimensional imagery is still in its infancy compared to human labeling. Human labeling (manual curation) is not feasible for large 3D datasets, especially when a single sample is often imaged many times, and each volume can contain several gigabytes of data. Moreover, previous approaches have scalability issues. The approach described in this paper provides a solution for all these problems using Dask.

## Dataset, Softwares, and Infrastructure for Research

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b569ab3553e1e9bcd42_architecture-1024x631.png" alt="Architecture in the form of a Graph.Browsers -> Jupyter Lab (jupyter lab server) and Dask ExtensionsJupyter Notebooks -> Jupyter LabPython kernel -> Jupyter Notebook, Local file system, Cloud storage, Dask Client (to Scheduler)Local file system, Cloud storage -> Dask workers (Zarr)Dask scheduler -> Dask dashboard, Dask workersObjects enclosed in a box: Dask, dask scheduler, Dask workers, Coield, AWS, NERSC">
<figcaption><em>3D microstructure interactive image analysis and visualization system architecture</em>.<br>Source: <a href="https://github.com/dani-lbnl/SC20_pyHPC/blob/master/SC20_pyHPC_dask4volumes.pdf" target="_blank">Article presented at Super Computing 2020</a></figcaption>

The data acquired from previous related work are available publically in a data collection of 6 terabytes in total, where each file contains about 60 GB and more than 14 billion voxels. That's a lot of data! This research uses one of the volumes in the data collection alongside synthetic datasets of about 200³ voxels generated to suit the need. The team uses Dask on a local machine (Dell XPS 13 7390), on a supercomputer (NERSC Cori), and on the cloud (AWS) with Coiled to analyze this data.

The dataset is stored in [Zarr](https://zarr.readthedocs.io/en/stable/) format using [xarray](http://xarray.pydata.org/en/stable/). Zarr is a format for storing chunked n-dimensional arrays, and XArray is a library for labeled arrays and datasets in Python. This chunked and compressed multi-scale pyramid storage allows parallel analysis and visualization with Dask.[Insight Toolkit (ITK)](https://itk.org) is used for its rich assortment of 2D and 3D image analysis algorithms. [itkwidgets](https://github.com/InsightSoftwareConsortium/itkwidgets) provides interactive Jupyter widgets to visualize images, point sets, and meshes in 3D or 2D. [Jupyter](https://jupyter.org) notebooks are the interactive IDEs used to write Python code. Finally, Dask is used for parallel computing as described in the following section.

## Dask and Coiled for Parallel, Distributed, and Cloud Computing

[Dask](https://dask.org/) is used to parallelize computation and accelerate the analysis. Dask schedulers are a core part of Dask. They take a task graph and execute it on parallel hardware, where the task graphs can also be specified as Python dictionaries. There are various schedulers with different performance optimizations ranging from single machine, single core to threaded, multiprocessing, and distributed.

Dask schedulers proved very useful for this research:

*"The Dask single-machine schedulers have logic to execute the graph in a way that minimizes memory footprint. As a result, our algorithms can be quickly extended on smaller datasets with a laptop or workstation. Then, the same code can scale to extremely large datasets by running Dask over a HPC cluster backed by MPI, a cluster with a scheduler like SLURM, or a dynamically-generated, cloud service running a Kubernetes cluster."*

[Coiled Cloud](https://coiled.io/cloud/) makes it easy to deploy Dask on the cloud. It takes care of all the DevOps involved in cloud computing and provides a simple interface that can be run on any machine to leverage hundreds of cores on an AWS cluster. Support for Azure is also in the works! In this showcase, Coiled Cloud was used to run the same computing environment on the cloud to compare runtime benchmarks.

## Coiled Makes Computation 6x Faster! 

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/633f7b569ab355609d9bcd41_comparison.png" alt="Laptop, Dell XPS 13 7390 2 workers, and 6 cores, takes 13 minutes to complete execution.NERSC Cori, 100 workers, and 2CPUs per workers, takes half a minute.Coiled cloud, 100 workers, and 1 CPU per worker, takes 2 minutes">
<figcaption>Source: <a href="https://github.com/dani-lbnl/SC20_pyHPC/blob/master/SC20_pyHPC_dask4volumes.pdf" target="_blank">Article presented at Super Computing 2020</a></figcaption>

The computation took **13 minutes on a standard laptop** and the same processing pipeline was executed on a **Coiled cluster in 2 minutes**. The paper notes how the same computing environment could run the JupyterLab server and client with custom JupyterLab extensions, the Jupyter Kernel process, as well as the Dask scheduler, workers, and dashboard.

The team also describes powerful similarities across computing environments: 

*"High-memory resources were not required – the limited memory capacities of the laptop and Coiled workers were sufficient for the large dataset. The user experience, i.e. interactive Jupyter workspaces accessed through the browser on the laptop, was exactly the same. And, most importantly, the same software was used without modification; to change the computational backend, the Dask Distributed backend was simply changed to a distributed cluster resource, which resulted in a 25X acceleration when processing the larger dataset."*

## Get the most out of your data science

We always love seeing Dask and Coiled being used to make data science accessible to the scientific community. This use case for scaling computer vision and 3D data analysis is a perfect demonstration of **Dask's flexibility and power**. You can check out their [code repository on GitHub](https://github.com/dani-lbnl/SC20_pyHPC) and try Coiled Cloud for free below!

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id=""hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec"" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kHV8YbTLoWQ90XdV7fVcQqltqaUtA&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=2074c6e6-465a-4d3b-9404-c1e756859df6&amp;redirect_url=APefjpHrUn5kocFLU0gLBRk5XnqplS91VnhkARJjpZKODHxfTr5CzvZdMonuyJZeznUR4kP5kLuB_fVHBCVdC73llahS5FQek0L8fEUUU_aUM_sbQnY7vRR1zYPu3T98VrGP7d9w0FIi&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Faccelerating-microstructural-analytics-dask-coiled&amp;ts=1744255019980" style="" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>