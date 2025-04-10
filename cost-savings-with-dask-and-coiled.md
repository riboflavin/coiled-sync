---
title: Cost Savings with Dask and Coiled
description: Coiled can often save money for an organization running Dask. This article goes through the most common ways in which we see that happen.
blogpost: true
date: 
author: 
---

# Cost Savings with Dask and Coiled

## Summary

Coiled can often save money for an organization running Dask. This article goes through the most common ways in which we see that happen. 

We start and end with the big abstract points of "people are expensive" and "you can move faster" and fill in the middle with lots of technical details.

## 1 – Personnel cost savings

First and foremost, running Dask in the cloud takes effort by smart people. Smart people are expensive. We often see data science groups spending half of an FTE maintaining the cloud environment where Dask is run. This includes activities like the following:

1. Setting up and tending EC2 / Kubernetes clusters
2. Managing docker images for the team
3. Tracking resource usage and shutting down things that shouldn't be running
4. Handling upgrades
5. Debugging issues from their teammates

Half-an-FTE often costs between $50k and $150k/year depending on their seniority. Allowing this person to go back to their day job is, often, the largest cost savings for groups just getting started. Learn more in our [Build vs. Buy page](https://coiled.io/build-vs-buy).

## 2 – Track and shut down idle resources

People leave things on, either through negligence (whoops! I didn't realize that I left this notebook open) through things slipping through the cracks (hrm, I definitely shut this down, but it looks like the signal never made it to the cloud manager) or through ignorance (what do you mean I had to clean up network resources as well as instances? We get charged for those separately?) 

Coiled tracks everything and provides a zero-cost default experience, allowing people to rest easier. If, somehow, something sneaks through, our pagers go off, not yours.

This changes the way we work. As Coiled users we often don't really care about being diligent shutting down clusters. We know that the automated systems will detect idleness and clean them up.

## 3 – Spot / Preemptible instances

Instances that can disappear are cheaper than instances that can't, often by a factor of 2-3x. By leveraging spot instances we can drastically reduce costs. However, using Spot is hard, especially when you need something to work. Coiled allows the following policies:

1. Prefer spot, but use on-demand if no spot instances are available

2. As spot machines are reclaimed

     a. Give Dask workers notice so that they can gracefully terminate, allowing smooth operation within Dask and no lost            work
     b. Backfill these slots with new on-demand instances

3. As spot instances become available again (work in progress)

     a. Gracefully retire on-demand instances
     b. Backfill with spot

Doing this dance well is hard, but it allows us to substitute an expensive resource with a cheap one with little degradation in experience. Cost savings here can be ~ 50%. Learn more in [our blog post](https://www.coiled.io/blog/save-money-with-spot).

## 4 – ARM

ARM chips are fast and cheap, often resulting in [20-30% cost reductions](https://blog.coiled.io/blog/dask-graviton.html). Coiled makes it easy to switch between instance types and try ARM on your existing workloads. Coiled also makes it easy to substitute out x86 Python packages for ARM packages without any effort in the common case.

This effort is new and exciting. We encourage users engaged with ARM chips to [try it out](https://docs.coiled.io/user_guide/clusters/arm.html)!

## 5 – Transparency

Coiled tracks everything that is going on and gives aggregated views to cost owners in an organization. This allows organizations to make informed decisions and focus their energies appropriately.

Transparency takes a few forms. You can view the cost of currently running clusters, as well as the cost of historical computations both started by yourself, and by others in your team. You can also dive in to individual clusters and see what operations and even what lines of code accounted for the largest part of the cost.  

You can also view aggregated cost information by user and by time to help identify opportunities for future improvement and cost assignment. See our [documentation on managing costs](https://docs.coiled.io/user_guide/costs/index.html) for more details.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/64d1345b6369f5e3f48c4829_Screenshot%202023-08-07%20at%2011.13.14%20AM.png" loading="lazy" alt="">

Account administrators can see a detailed breakdown of Coiled usage by a number of dimensions like user, GitHub reference, or tag.

## 6 – Limits and bounds

We also make it easy to bound the amount that you spend with us both through sensible tagging of cloud resources, and through our own limiting controls.

#### Cloud limits

Each resource that Coiled creates will be tagged with **owner: coiled, **allowing you to see all the resources** **created by your account. Alternatively, you can set custom tags for each cluster created so you have better visibility into cost breakdown. Refer to the [section about tags](https://docs.coiled.io/user_guide/costs/tags.html) in the documentation.

This allows you to more easily set alerts in your cloud provider account.

When you set alerts, AWS or GCP will send you an email notification when you have reached a certain threshold, so you can know how much you have spent so far. You can follow the official documentation on how to set up billing alerts:

- [Create a billing alarm to monitor your estimated AWS charges](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html)
- [Protect your Google Cloud spending with budgets](https://cloud.google.com/blog/topics/developers-practitioners/protect-your-google-cloud-spending-budgets)

#### Coiled usage limits

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/6364208b390aec411e6fe850_uAzCwgM9ovjlPQFuj2QEd_BbrT5fZXBOJtojFMiJAy5hinbLuaC7OyoZj8o-0Quln8R8I3MZ2xVr9yRr8uo7Y9CvMyM6RY2RYnFYTy9IIITJNu46tu8cpOasYo6eDf2Ru6pu-QnmU2Nul8HW-3BPUmIw35XVObRAIAMrnkFM4hI_h8RKEjulw5yxXJvmxQ.png" alt="" loading="lazy">

Additionally Coiled can help you bound how much you or your teammates spend with us. Coiled offers the following bounds:

1. A monthly total spending limit
2. Instantaneous spending limit on the full account (set by Coiled)
3. Instantaneous spending limits on each user (set by account admins)

These allow users to bound their overall spend, as well as enforce bounds on individual teammates. See our [documentation](https://docs.coiled.io/user_guide/users/index.html) to learn more.

## 7 – Risk and support

Coiled is maintained by a professional team and monitored 24x7. The risk associated with such a platform is lower than a DIY approach. This team understands cloud infrastructure, Dask, and the underlying computational libraries well enough to provide a more consistent and predictable experience for organizations that rely on this technology stack for business-critical workloads.

## 8 – Sensible configuration

There are many other small configuration choices that we make in our cloud architecture that each have non-trivial effects. These compound to significant savings. We'll list a few below:

1. Good default instance choices, particularly those in the t3 line, which support cheap burstable compute
2. Avoiding NAT gateways which are common causes of surprising data transfer bills.
3. Multi-Zone worker pools, allowing users to get the best availability possible for Spot or rare instance types, even when supply is low
4. …

These choices are technical and in the weeds, but can each have a strongly positive impact on overall costs.

## 9 – Opportunity costs and time to delivery

You've opted to use the cloud because you value moving quickly.

The median Coiled user spends low thousands of dollars per month. While real money, this isn't that big of a deal related to the value of moving faster. Coiled empowers teams to shave several months off of their timelines. When compared to the costs of monthly burn and waiting to build something internally, the costs here are often trivial in relation.

## Conclusion

Coiled is free to try, up to about $500/month of managed cloud usage. Many of our customers pay only tens of thousands of dollars per year, far less than they would pay trying to build such a system themselves. Coiled provides a cost-constrained experience that also delights data science and data engineering users, making them more productive with their time.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id=""hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec"" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kF8cJ14_NFCXUtefIroMFK9yuGCPQ&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=156d741b-9935-442a-902a-5968642b26c1&amp;redirect_url=APefjpE4H0H8bUCFxLILVhfu5rIOpwqhsBtSDEsoYKTqmo4od3DcgNxLTo-FWvHNEC3tgzLc57hnaF-qP93ID5-gS_OXcNUYtB1LOyt4htuVi4MCNNDRnU2BYGgE9qceiVPHNkYfCoRW&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fcost-savings-with-dask-and-coiled&amp;ts=1744255827272" style="" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>