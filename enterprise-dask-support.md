---
title: Enterprise Dask Support
description: Along with the Cloud SaaS product, Coiled sells enterprise support for Dask. This article will be a more informal description of how we operate and why.
blogpost: true
date: 
author: 
---

# Enterprise Dask Support

Along with the Cloud SaaS product, Coiled sells enterprise support for Dask. Mostly people buy this for these three things:

1. Office hours with Dask experts
2. Private GitHub issue tracker
3. Commitment to quickly fix Dask bugs that affect critical workloads should they arise

Support contracts also come with a variety of other things, like indemnity insurance, response time SLAs, and so on, but these are usually mostly to satisfy procurement checklists rather than drivers on the business side.

Our contracts list everything out in detail, but I often find myself explaining to customers how things work and why we structure things the way we do. This article will be a more informal description of how we operate and why.

### Who needs Enterprise Dask Support?

This tends to be larger companies that are moderately far along in their Dask journey. They usually have a few groups using Dask and it's found its way into critical workloads. Often Dask has spread throughout the organization faster than their Dask expertise has grown, and so there are a handful of Dask experts who are somewhat overburdened trying to keep everyone happy and all critical workloads humming nicely.

We charge mid-six-figure annual deal sizes for our support contracts, so these are usually larger companies.

### How deep do you get?

We need to strike a balance between getting deep enough with your problem to understand how to help, while staying distant enough that we focus on Dask (what we're good at) rather than getting sucked into internal customer systems. Usually the customers who want enterprise support contracts are pretty sensitive about having us deeply in their systems, so everyone is aligned with finding good boundaries of communication.

As a result, we do not offer "hands-on-keyboard" support, where we would go into your company, look at your data, and build ML models / data engineering pipelines for you. Honestly, we'd be pretty bad at that anyway. It's not our skillset. When folks do need this level of support we bring in consultants that we know and trust. Most often we recommend Quansight who are great at general purpose open source Python work.

Instead, we find that a combination of a few office hours a month, an issue tracker, and all-you-can-eat OSS bugfixing works well in combination. Let's talk through these three below:

### 1 – Office Hours

It's useful to talk to an expert. Many of the problems that our users face can be resolved by pointing them to a project or API of which they weren't aware. It's common that the right 30 minute meeting can accelerate weeks of work.

Contracts typically include a finite number of office hours per month, somewhere between two and ten. Folks can use these however they like, but a typical allocation of ten hours looks like the following:

1. One open office hour every week (4hrs)
2. A lunch and learn style lecture on a topic facing the group every month (1hr + 1hr prep)
3. A deeper dive on a harder problem encountered by some internal team, often including screensharing going through performance dashboards, logs, etc. (2hr)

These hours are designed to uncover issues and transfer knowledge. Often they result in issues raised for work upstream. This is where the bulk of the value comes.

### 2 – Dask Critical BugFixing

While office hours are finite and precious, Dask bugfixing is managed in more of an "all-you-can-eat" fashion. Often in office hours we'll find that Dask was underperforming in some way (Dask is far from perfect) and we can raise this as an issue to be resolved upstream.  

Once we've found work to do on the public open source project we get access to a much larger team to work on your problems. As a company we're mostly focused around discovering high impact work, prioritizing that work, and executing it swiftly. If we can solve your problem by enhancing Dask then all of our incentives are very well aligned.

If you think in terms of hourly rates, the office hours are expensive (thousands of dollars per hour) while the Dask OSS work is very cheap (often $100 per hour or less). This reflects our perception of costs and our priorities. We spend the office hour time figuring out how to align our company and yours, and then we operate more as partners with a shared goal of improving the open source software.  

Of course, even this isn't entirely free. Our OSS team is large and skilled, but still finite. We also prioritize based on importance across users. Financially, we're incentivized to provide enough value to customers so that they want to renew at the end of the year. The larger the contract, the more value we know we need to provide.  

### 3 – Private Github Issue Tracker

We need a way to communicate on the ongoing technical work. We find that Github is the easiest and least distracting way to do this. Being outside of the open source tracker provides some privacy for sensitive communication, and also makes it easier for us to prioritize and track things without them getting lost.

We also jump on video calls on an as-needed basis and, very rarely, have slack channels open with customers. We've found that Github works best though to keep things private and also well-tracked.  

### What we don't do

We're very efficient at diagnosing Dask issues, but inefficient at diagnosing issues related to customers' internal systems. This comes up in a few scenarios:

1. "Pods in our in-house Kubernetes system aren't coming up. It says that it can't load the image. Why?"

   If we don't manage your Dask deployment solution then we really can't do much to help you here. Probably we don't even have the permissions to diagnose this. We encourage you to try out Coiled's deployment solution.  It's pretty slick.

2. "Our data scientist ran this giant notebook and it's slow. Can you make it faster?"

   This may have more to do with your code than ours. Your team is likely more efficient at figuring out where the issue is than we are, and they're a lot cheaper.

   However, we could help here in a few ways:

   a. Getting on a call and looking at Dask diagnostics together, helping to pinpoint what might be going wrong.
   b. Teaching your team more about Dask diagnostics and how to use them to identify performance issues.
   c. Providing general Python performance advice during office hours.

   But we need to be clear that custom workload performance is out-of-scope for the all-you-can-eat model. We can provide advice and training, but ultimately custom workloads are the customer's responsibility. If the customer is able to provide a minimal reproducer of the issue or benchmark then we can more effectively take on ownership.

3. "We've made an in-house library around Dask to train models. Can you help us to maintain it?"

   We would likely try to direct you to open source technologies that might do similar things. Alternatively, we could connect you with excellent consultants who can be more hands-on.

4. "We've found an issue in dask-kubernetes/dask-gateway/dask-cloudprovider. Can you fix it?"

   This invariably leads us into Problem #1.  We are happy to go over configuration on these tools during office hours, but don't include them in our all-you-can-eat structure. We've found that they're hard to support in deep enterprise cases. This is why we built the Coiled deployment platform.

5. "We have many new people to Dask, can you answer all of their questions on Slack?"

   Sure, let's use an office hour or two and give some broad training. However, we can't be available for lots of novice Q&A. Instead, we ask that your company selects a small number of folks to route these questions through to us. We find that those folks are usually pretty Dask savvy and are able to provide basic level-1 support. Please think of us more as L2/L3 support.

### Limited Seats

As the last question implies, we typically limit the number of folks that we talk to in an organization. This tends to be 1-2 people per project (companies paying several hundred thousand annually typically have a few groups using Dask). These people become deep experts and help to buffer us from lots of small problems. As Dask expands in the organization and we add more projects we also add more contact seats and typically start to ask for more money (which is often fine, because the new projects often come in with new budgets).

### Sometimes, we also do new development

Sometimes companies come to us with paid feature development requests like "Hey, can you integrate Dask with Library X?". We consider these, but reject more than we accept. We become much more excited if these are already on our roadmap, and the company wants to help us with design and testing as more of a partner.

### Aligned Incentives

Over the years Dask has been funded by a wide variety of mechanisms. I really like the support model laid out above. It does a good job of delivering solid value to customers, while also aligning incentives to help make the project better. Everyone focuses on their core skillset and generally has a good time.

Dask has been developed in this manner for many years. It would not be nearly as pragmatic without the valuable support and technical insight of its support partners throughout the years.

<span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id="hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_8e4f34db-efc2-457d-b57b-19c8363d59d5" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=8e4f34db-efc2-457d-b57b-19c8363d59d5&amp;signature=AAH58kGd9Q8rTqqE1EfOyOU969jbhWNJnA&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=070d9128-c6bb-4d84-8797-d88c30a1d481&amp;redirect_url=APefjpEBOpDEAd1hGuebTyg4uChunnS0QQKOd1HZAlObL3iwbJDQXdMkvGt2TB-C7bsp3_XjH1rUizzHoCD7ZQjp1uTy9-TM47LWhexSjwBj_bM6CCQxZ2l3S-y2yFDt2zyA7o1dpuN8&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.coiled.io%2Fblog%2Fenterprise-dask-support&amp;ts=1744255825924" style=" cta_dest_link="https://www.coiled.io/product-overview" title="Learn More About Coiled"><div style="text-align: center;"><span style="font-size: 16px; font-family: Helvetica, Arial, sans-serif;">Learn More About Coiled</span></div></a></span>
</span>