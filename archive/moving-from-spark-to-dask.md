---
title: Spark to Dask: The Good, Bad, and Ugly of Moving from Spark to Dask
description: Spark is no doubt a fast analytical tool that provides high-speed queries for large datasets, but recent client testimonials tell us that Dask is even faster.
blogpost: true
date: 
author: 
---

# Spark to Dask: The Good, Bad, and Ugly of Moving from Spark to Dask

Apache Spark has long been a popular tool for handling petabytes of data in analytics applications. It's often used for big data and machine learning, and most organizations use it with cloud infrastructure to run models and build algorithms. Spark is no doubt a fast analytical tool that provides high-speed queries for large datasets, but recent client testimonials tell us that Dask is even faster. So, what should you keep in mind when moving from Spark to Dask?

We interviewed [Steppingblocks](https://www.steppingblocks.com/), an analytic organization that provides valuable insights on workforce and education outcomes based on 130 million individuals registered in US universities, employed at private organizations, and working at government agencies. Their data journey started with basic linear SQL, they moved to Spark, and later they migrated to Dask. By moving to Dask, Steppingblocks shortened their processing times from 24 hours in Spark to 6 hours in Dask, but they learned several lessons in the process. They've provided us (and you!) with some valuable insights on the good, bad, and ugly should you decide to migrate your organization's machine learning and big data processing over to Dask.

## Spark to Dask: What Prompted the Move?

For most organizations, migration is a huge undertaking that isn't decided lightly. You need a return on your investment, and Steppingblocks was no different. They saw some improvements when moving from standard relational SQL to Spark, but they also found several challenges with Spark development that motivated their team to look elsewhere.

The biggest challenge was debugging code in Spark. Developers knew Python but did not know Spark, so it was a learning experience that didn't come with much support. They found that traceback errors were not very intuitive, and they spent too much time on Stack Overflow to find answers to their problems. Support from Spark was too expensive, so any problems were left to their own devices. Finally, Spark felt like overkill for what they were doing, and it showed in their computing costs. Costs and resources were high for what Steppingblocks needed, so they turned to [Coiled](/old-home-4) to find out if it could better support their business requirements.

Also, Steppingblocks was purely a Python development team. They found that they were trying to force-fit their Python into the Spark framework. What attracted them to Dask was the more native way it worked with Python, and it made development simpler.

## The Good That Led to Migration Success

Every migration takes planning and work, but not every migration is quick and painless. Before getting into Steppingblocks challenges, they revealed several successes that can be attributed to Dask. First, let's talk about the migration successes.

The biggest win was the improvement in ETL speed. Steppingblocks processing time went from 24 hours to run their ETL procedures to 6 hours. Integration of MLFlow models with Dask was much more efficient with better feedback from the Dask system should they run into any errors from their Python code. The Coiled team was supportive and helped them work through difficult errors that developers could not figure out on their own.

Running ETLs in Spark requires cloud resources, but Dask can run locally. With Spark, you need a powerful machine only available in the cloud even for small datasets and computing tasks. Developing on Spark with datasets of any size requires access to cloud resources.  By contrast, the ability for developers to run Dask locally accelerated the development lifecycle.

Overall, Steppingblocks was able to migrate 90% of their codebase from Spark to Dask in only a few weeks with a few extra months needed for optimization. The entire initial migration was done in 3 months. They shared a few success metrics below:

* Timeframe: **Migration took three months** with two engineers and 9 months for optimization, but optimization delays were not related to Dask.
* Cost: The **cost to run Dask is 40% less than Spark**.
* Run time: **Dask tasks run three times faster than Spark** ETL queries and use less CPU resources.
* Codebase: The **main ETL codebase took three months to build** with 13,000 lines of code. Developers then built the codebase to 33,000 lines of code in nine months of optimization, much of which was external library integration.

<img src="https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/6372d8132bea68602fa92c08_Dask%20v%20Spark%20benefits.png" loading="lazy" alt=">

## The Bad and Ugly, or Challenges during Spark to Dask Migration

Scaling Dask was a challenge at times. Steppingblocks developers could scale Dask locally but then production Dask servers would return errors. This is where [Coiled support](https://docs.coiled.io/user_guide/support.html) helped Steppingblocks work through their errors and figure out what was wrong. This type of support is costly for commercial companies developing with Spark for enterprise applications.

The biggest challenge was porting Spark SQL code to Dask. It's what took the most time, including the time to optimize it. As with any new environment, developers need time to learn the new framework and grow accustomed to changes.

## Migration Aftermath and the Dask Journey Since Deployment

As any developer knows, initial migration doesn't stop when code is finally ported. You still need to maintain and reliably work with the new framework. Moving away from linear relational SQL is always a benefit for machine learning and modeling, but choosing the right platform is the challenge to ensure that you have the right tools and environment.

Steppingblocks found it much easier to migrate machine learning models and iterate them more rapidly. Their developers work in a fast-paced environment, and Dask supports their efforts. Coiled also works quickly, so developers must ensure that they are aware of the latest updates and changes to Coiled versions during testing and deployment. Whenever they have issues, however, the Coiled team helps support them.

Because Dask can work locally, developers can test their Python code in all three stages – local, local Dask, and Coiled. Moving Python modules rather than notebooks makes development iterations much faster and more reliable. Developers can build software following industry best practices including linting, unit testing, and integration testing.

## Moving from Spark to Dask? Here are Lessons Learned and Tips and Tricks

If you're thinking about moving from Spark to Dask, the Steppingblocks developers have some tips and tricks from their own lessons learned. You might find that you have your own challenges based on your unique business requirements, but one consistent story from developers is that the Coiled team was helpful in migration success and will be there to help work through any issues.

Here are a few tips before you plan out your migration from Spark to Dask:

* You must run mirror development environments locally and on the cluster, or you may run into errors from version differences, but the newly released Coiled Runtime helps alleviate this issue.
* Beware of defaults around partitioning and prepare to split out, but it's useful to take advantage of map_partitions to help reduce issues.
* Use worker.data in Dask for lookup tables and don't forget to leverage caching for speed.
* Don't rely on the cloud for 100% uptime. Instead, build defensive programming techniques for retries in case of timeouts.
* Leverage distributed.wait to break down large computer graphs if memory usage becomes an issue.

Watch the recording of our webinar with Sébastien Arnaud of Steppingblocks: ["Why We Switched from Spark to Dask | Our Data Journey"](https://www.youtube.com/watch?v=jR0Y7NqKJs8).

<iframe allowfullscreen="true" frameborder="0" scrolling="no" src="https://www.youtube.com/embed/jR0Y7NqKJs8?enablejsapi=1&amp;origin=https%3A%2F%2Fwww.coiled.io" title="Spark vs Dask | Why We Switched from Spark to Dask | Sébastien Arnaud at Steppingblocks | June 2022" data-gtm-yt-inspected-34277050_38="true" id="298436322" data-gtm-yt-inspected-12="true"></iframe>