---
title: How Coiled sets memory limit for Dask workers
description: Having Dask workers die from memory overuse is common, so we thought that we'd investigate this further.
blogpost: true
date: 
author: 
---

# How Coiled sets memory limit for Dask workers

## Problem

While running workloads to test Dask reliability, we noticed that some workers were freezing or dying when the OS stepped in and started killing processes when the system ran out of memory.

Having Dask workers die from memory overuse is common, so we thought that we'd investigate this further.

## Background

Dask manages memory, so it needs to know how much memory it has available. 

Setting the memory limit correctly is important to the performance of your Dask cluster. This memory limit affects when Dask spills data in memory to disk, and when it restarts workers that are out of memory.

When the memory limit is set **too low**, Dask won't use as much memory as it could. That's not the worst thing, but it means your work will potentially take longer than it needs to.

When the memory limit is set **too high**, Dask can fail to spill to disk when needed and fail to restart workers when they're out of memory. This is bad because if a worker isn't killed when it's out of memory, then it can freeze and potentially your cluster will be stuck waiting for the work from this worker. If the worker is restarted, then Dask will retry the work.

Dask attempts to determine the correct memory limit to use. It does this by looking at the **total system memory** (including virtual memory) and the **cgroup memory limit** if set. (Dask also looks at [RSS rlimit](https://docs.python.org/3/library/resource.html#resource.RLIMIT_RSS) if set). You can also explicitly set the memory limit. Dask then uses the minimum of these values.

The upshot is that in many deployments—including how we had been deploying Dask with Coiled—the memory limit will be set to the total system memory. That's not ideal because that limit is **too high**: we're also running an OS and logging agents.

## Solution

To address this, Coiled now sets the **cgroup memory limit** on the containers running dask. We set this to the available memory on the system **_before_** Dask starts running.

Here's how we've implemented this.

First, our script that starts Dask on each VM now sets an environment variable with the available memory:

<a href="#" class="copy-button button-2 w-button" fs-copyclip-element="click" fs-copyclip-message="Copied" fs-copyclip-duration="3000">Copy</a>
<pre class="language-python"><code class="language-python hljs" fs-copyclip-element="copy-this">export AVAILABLE_MEMORY_FOR_DASK=$(grep -i memavailable /proc/meminfo | sed 's/[a-zA-Z]+:\s+//')
</code></pre>

Then, we've modified the Docker compose file we use to start Dask (we run Dask in containers, and configure the containers using Docker compose). We've added a line to explicitly set **mem_limit** for the container to ${AVAILABLE_MEMORY_FOR_DASK:-0}.

We could have just used $AVAILABLE_MEMORY_FOR_DASK, but instead we're setting a default value of 0 (i.e., no limit) just in case the environment variable isn't set. Mostly that's unnecessary, but it makes deployment safer because the script that sets the environment variable is part of our custom AMI while the Docker compose file is created for each cluster by our control-plane.

With a Dask client connected to your Coiled (or other Dask) cluster, you can check the value of the memory limit Dask is using by running:

<a href="#" class="copy-button button-2 w-button" fs-copyclip-element="click" fs-copyclip-message="Copied" fs-copyclip-duration="3000">Copy</a>
<pre class="language-python"><code class="language-python hljs" fs-copyclip-element="copy-this">client.run(lambda dask_worker: dask_worker.memory_manager.memory_limit)
</code></pre>

The upshot is that since we've made this change, Dask running on Coiled now does a better job spilling to disk and restarting workers when appropriate, and we're seeing increased reliability from our testing.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id=""hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec"" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_be9599a4-6c30-4743-aa64-3e10e4ddc6b6" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=be9599a4-6c30-4743-aa64-3e10e4ddc6b6&amp;signature=AAH58kEAZXoCK57rNubyxdVKJOHBDT8bPA&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=479c6476-3533-40fd-9a9b-cad6b295b88e&amp;redirect_url=APefjpHFXTYV2PZCa2wKr7dqpI2AxMyNvYw4KSiMK8sAXPeLpl9yN8eftABR46Y1cCiG2-BXGA6YEWDnR1Dbitt9UIynyavQvUUMmmq_04U5P0m0hVaid2Q&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fhow-coiled-sets-memory-limit-for-dask-workers&amp;ts=1744256062819" style="" cta_dest_link="https://cloud.coiled.io/signup" title="Try Coiled"><div style="text-align: center;"><span style="font-family: Helvetica, Arial, sans-serif;"><span style="font-size: 16px;">Try </span><span style="font-size: 16px;">Coiled</span></span></div></a></span>
</span>