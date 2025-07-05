---
title: Use Mambaforge to Conda Install PyData Stack on your Apple M1 Silicon Machine
description: Running PyData libraries on an Apple M1 machine requires you to use an ARM64-tailored version of conda-forge. This article provides a step-by-step guide of how to set that up on your machine using mambaforge.
blogpost: true
date: 
author: 
---

# Use Mambaforge to Conda Install PyData Stack on your Apple M1 Silicon Machine

## **Use Mambaforge**

Running PyData libraries on an Apple M1 machine requires you to use an ARM64-tailored version of conda-forge. This article provides a step-by-step guide of how to set that up on your machine using **mambaforge**. 

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kHxSsPB8LrVycUWlHC83A0VK96QjA&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=fd59d181-5e3b-4366-8d74-0054dbb43599&amp;redirect_url=APefjpEPKIQKDsTgB-qjNguvsOlAdoD1dfkRWSgjKpCO9x7TD8gvXRC8rJH92-Fw9WbxWlslzE2JYQCegAE0EJxIOeFp3KYCeQhgdwZqbiZNaoDzr7TpFtQacghIMlaeCxl4KuYHmLzH&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fapple-arm64-mambaforge&amp;ts=1744255823778" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>

*Note: this article was first published in August 2021. Some of the issues with conda described below have since been resolved*. *I still find that **mambaforge** builds environments and installs packages* *faster and recommend trying it out.*

The Apple M1 silicon chip is a fantastic innovation: it's low-power, high-efficiency, has great battery life, and is inexpensive. However, if you're running the PyData Stack with default builds, you may encounter some strange behaviour.

When you install a library written for Intel CPU architecture, the code will be run through the Rosetta emulator. This often works great but may cause unexpected problems for specific PyData libraries. For example, I personally encountered this with Dask's memory usage blowing up when importing a <100MB dataset.

Since the internals of Rosetta is proprietary, there's no way to know exactly why this is happening. 

## Installing Python Libraries using Mambaforge

But fear not, there *is* a way forward. We can resolve much of this strange behaviour by installing ARM64-native versions of the PyData libraries. While we typically recommend Anaconda/miniconda for most installations, currently, the **mambaforge** deployment seems to have the smoothest ARM installation process. 

Below is a step-by-step guide to installing this version of conda and using it to install your favorite PyData libraries.

**1. Check your version of conda**

- To check your conda installation, type conda info in your terminal and check the value of the platform key. This should say osx-arm64.
- If it doesn't, go ahead and remove the folder containing your conda installation. Note that this will remove your software environments, so be sure to have those saved as .yml or .txt files. You can export the 'recipe' for an active software environment by typing conda list — export > env-name.txt OR environment.yml into your terminal.

**2. Download the mambaforge installer**

- Navigate to [***this Github repository***](https://github.com/conda-forge/miniforge) and select the Mambaforge-MacOSX-arm64 installer. You can, of course, also opt for one of the other deployment versions listed on that page.

**3. Run the installer.**

- This should ask you if you want it to run conda init for you. Select "yes". 
- If for some reason it doesn't do so (which happened in my case), you will have to run conda init yourself using /path/to/mambaforge/bin/conda init zsh. Note the zsh is only if you're working with the zsh shell. Use other shell flags as needed.

After restarting your terminal, you should now be able to use conda as you always have. You can, for example, begin by rebuilding your software environments using conda env create -f <path to environment.yml>.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kHxSsPB8LrVycUWlHC83A0VK96QjA&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=fd59d181-5e3b-4366-8d74-0054dbb43599&amp;redirect_url=APefjpEPKIQKDsTgB-qjNguvsOlAdoD1dfkRWSgjKpCO9x7TD8gvXRC8rJH92-Fw9WbxWlslzE2JYQCegAE0EJxIOeFp3KYCeQhgdwZqbiZNaoDzr7TpFtQacghIMlaeCxl4KuYHmLzH&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fapple-arm64-mambaforge&amp;ts=1744255823778" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>

## Specific issues when running Dask, Coiled and XGBoost

Below are some further notes on issues I encountered while trying to run a workload using Dask and XGBoost on Coiled:

```python
import coiled
from dask import Client
import dask.dataframe as dd
cluster = coiled.Cluster()
client = Client(cluster)
ddf = dd.read_parquet("<path to 100mb file on s3>")
ddf = ddf.persist()
```

I encountered three issues and would like to share my solutions for them as I can imagine others might be running into this as well:

1. Dask's memory usage was blowing up when importing a small  (<100MB) dataset with an error like the one below:

```
distributed.nanny — WARNING — Worker exceeded 95% memory budget. 
Restarting
```

2. In some software environments, my kernel was crashing when calling **import coiled**
3. After switching over to the mambaforge deployment, I could not install XGBoost.

**1. Dask**

The first issue was resolved by switching over to mambaforge and installing all the required PyData libraries from there, as outlined in the step-by-step guide above.

**2. Coiled**

The second issue was due to a conflict caused by the python-blosc library. For some unknown reason, kernels crash on Apple M1 machines when importing python-blosc (e.g. as part of import coiled) with a segmentation fault. To avoid this, make sure that python-blosc is not installed in your current software environment. If it is, run conda uninstall python-blosc to remove it.

**3. XGBoost**

Finally, If you're looking to run XGBoost on your Apple M1 machine, you'll need to jump through some extra hoops as there is currently no arm64-compatible version of XGBoost available on conda-forge. This is not a problem because XGBoost itself can actually run on your Apple M1 machine; the issue is with the dependencies that XGBoost installs, specifically numpy, scipy, and scikit-learn. These dependencies need to be arm64-compatible, which we'll ensure by completing the following steps:

1. Create a new conda environment containing Python, numpy, scikit-learn, and scipy. It's crucial to install these dependencies using conda and not pip here.

```
conda create -n boost
conda activate boost
conda install python=3.8.8 numpy scipy scikit-learn
```

2. Install the necessary libraries to compile XGBoost

```
conda install cmake llvm-openmp compilers
```

3. Now that we have the right dependencies in place, we can install XGBoost from pip. This is fine since we have all the right arm-64 dependencies installed already.

```
pip install xgboost
```

4. Take it for a spin!

Note: the steps above were summarised from [this great tutorial](https://towardsdatascience.com/install-xgboost-and-lightgbm-on-apple-m1-macs-cb75180a2dda) by [Fabrice Daniel](https://medium.com/u/926442548db0) and updated after helpful input from [Isuru Fernando](https://twitter.com/isuru_f).

## Moving Forward

As with any technological breakthrough, there are going to be some birthing pains as the industry recalibrates and adjusts to ripple-waves caused by Apple's switch to M1 silicon. We're excited to see companies like Anaconda and QuanStack and the Conda-Forge community continue to improve support for ARM hardware and stoked for the powerful future of PyData this promises!

We hope this post helps get your PyData workflows running smoothly and effectively on your Apple M1 machine. If you have any feedback or suggestions for future material, please do not hesitate to reach out to us on [Twitter](https://twitter.com/coiledhq?lang=en) or our [Coiled Community Slack channel](https://join.slack.com/t/coiled-users/shared_invite/zt-hx1fnr7k-In~Q8ui3XkQfvQon0yN5WQ).

If you're interested in trying out Coiled, which provides hosted Dask clusters, docker-less managed software, and one-click deployments, you can do so for free today when you click below.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kHxSsPB8LrVycUWlHC83A0VK96QjA&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=fd59d181-5e3b-4366-8d74-0054dbb43599&amp;redirect_url=APefjpEPKIQKDsTgB-qjNguvsOlAdoD1dfkRWSgjKpCO9x7TD8gvXRC8rJH92-Fw9WbxWlslzE2JYQCegAE0EJxIOeFp3KYCeQhgdwZqbiZNaoDzr7TpFtQacghIMlaeCxl4KuYHmLzH&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fapple-arm64-mambaforge&amp;ts=1744255823778" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>