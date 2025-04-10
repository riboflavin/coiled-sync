---
title: Coiled, one year in
description: I wanted to take this opportunity to talk a little bit about the journey since starting Coiled, and what I think comes next...
blogpost: true
date: 
author: 
---

# Coiled, one year in

I wanted to take this opportunity to talk a little bit about the journey since starting Coiled, and what I think comes next...

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/634880e98041414d2a59bba2_image-2-1024x292.png)

## History: building a company from an existing community

January last year I (matt) left NVIDIA to start Coiled.  This was an interesting and challenging process in which I received a lot of help.  This section chronicles that journey in three phases, and should be a good peek into what starting a company looks like for anyone interested in the process.

### January-February — Company size: 1

I spent most of this period meeting with many different kinds of people

1. Other entrepreneurs who knew what to expect
2. Lawyers to help incorporate the company
3. Potential anchor clients to help us bootstrap
4. Potential first-hire employees to both help build out the company, and then run parts of it
5. Investors to explore if we wanted to take venture or angel funding

I found that I had to make lots of high impact decisions on very little information.  This was an uncomfortable process, but in the end, things mostly worked out.  After you talk to a few people in any category (like lawyers, potential employees, clients, investors) you get a sense for what you're looking for, and can usually make a sensible decision.

This period ended with the [Dask Developer Summit](https://web.archive.org/web/20220619122148/https://blog.dask.org/2020/04/28/dask-summit), a great reminder that I wasn't alone in this process, and then the beginning of COVID-19, a less-than-great reminder that business success relies on adapting to changing circumstances.

#### Entrepreneurs

Special thanks to folks like Travis Oliphant/Peter Wang (Anaconda), Wes McKinney (Ursa), Andrew Montalenti (Parse.ly), Jeremiah Lowin (Prefect), Rodrigo and Felipe Aramburu (BlazingSQL) and many others who helped out both logistically and emotionally.

#### Employees

Most of Coiled's current leadership team was assembled during this period (Rami, Hugo, James) while others were picked up later on in the process (Scott). 

There were also some mis-steps.  Very early on there was an engineer who built out a product prototype on his own time in order to see if we would work well together.  This ended up not working out, which caused mutual stress.  There were also early hires that ended up not being a good fit.  We found and used contractors for various functions, some of which worked well and some which didn't.  In general, I learned that expecting and dealing with failure is probably more important than learning to avoid it.

#### Investors

My original plan was to bootstrap.  I personally know many large companies that use Dask today that were happy to be early clients.

However, it quickly became clear that VCs valued this nascent company at levels that, to me, seemed silly to pass up at the time.  We ended up taking $5,000,000 in seed funding, giving up 20% of the company while retaining all control (the only thing I couldn't do was pay myself an unreasonable salary).  I ended up taking this deal, which in hindsight was great.  It allowed us to hire people with a wide range of risk tolerances, and allowed for a sensible work-life balance.

### March-August — Company Size: 6

So we hired some engineers, a marketing guru, and went to work.  This was a very small team relative to the funding that we had, but it worked well.  An engineering team of four can be incredibly efficient.  We had a prototype up in June, and launched in the beginning of September.

At the same time there was a lot of client negotiation and brand building. 

#### Client negotiation

We had good momentum in February with a few large financial clients in February.  In March all momentum was put on hold due to the COVID-19 outbreak.  Eventually COVID became a boon to technology companies, but back in March everyone was concerned about a financial crisis, and so spending in many organizations froze.

By the summer things had thawed, and I was spending a good deal of time in business and legal negotiations.  I have some experience working on the business side of deals, but almost no experience navigating the legal and internal corporate politics of enterprise sales.  At this point we found Scott, who joined us as a sales advisor.  This made a world of difference.

#### Renewed focus on smaller teams

At the same time we were also talking to some much smaller groups using Dask and wanting to use Coiled.  These conversations were much more similar, and required an order of magnitude less work.   In general the person that we found ourselves talking to was …

* A team leader of a data science / machine learning team of 2-10
* 6-12 months in to using Dask
* Had successfully used it on a big machine
* Had then stood up some cloud infrastructure, usually with Kubernetes to scale out
* Realized quickly that they definitely didn't want to do this.  This was taking 25-50% of an FTE, and wasn't even that good of a solution.
* Coiled's lightweight platform did everything that they wanted, and was exactly the user experience that they were looking for
* They're happy to double their current Dask spending, typically around $3000-5000 / month in order to remove the DevOps pain, and hopefully become more efficient cost wise as well

These deals were not nearly as lucrative as the larger enterprise deals, but they happened quickly, and resulted in us improving our product much more quickly.  Coiled's engineering team transitioned from "doing what Matt thought was a good idea" to "doing what several customers were all telling us in unison".

#### Spreading the Word

During this time, we also wanted to make sure that people could find us. To this end, we built out the MVP of our Marketing and Evangelism engines. This included our website, mailing list, a consistent blog with a consistent cadence, and a variety of social channels (Twitter, LinkedIn, YouTube, Medium). We also collaborated with broader PyData activities by appearing on podcasts, speaking at conferences, co-marketing with partner organizations, and presenting at Meetups.  These activities then lead to tight feedback loops between Product, Marketing, and Sales, which helped us when we [launched in late September.](https://web.archive.org/web/20220619122148/https://medium.com/coiled-hq/coiled-dask-for-everyone-everywhere-376f5de0eff4)

### September-December — Company Size: 15

Things were clearly working at this stage.  We had signed a couple of the large enterprise contracts, had a bit less than $1M coming in the door, and had about 1000 signups on the public cloud service (only about 5-10% of which do meaningful work). 

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/634880e83dcf6cf8ed4754d5_qqDh2KvI5IXYPmyZyb5IYmSqCSFwFDvzhzFrqOAPXJ-n7X3zgh7wzKqkqu0htNN7JNdje8HOl2RuxZiBslMcHwNugIZRXKb3K0jrW3Owinrez4gDsJBbAFDraMF-feAjtdYzYvIG.png)

We were also slammed.

Everyone was busy, and so we decided to grow.  Hugo/Marketing hired a couple of people to take the day-to-day burden off of him.  Scott/Sales hired our first inside sales rep.  Engineering went from four people to ten. 

A lot of this period was about going from a small nimble team where everyone knew what was going on to building out a machine.  Today if you sign up for and spin up a cluster on Coiled there are now automatic signals that go to Marketing and Sales.  You'll get a ping from Oren (our inside sales rep) pretty soon after asking if you'd like to talk.  Some fraction of these turn into meetings with Scott/Sales, and some fraction of those turn into a commercial partnership going forward. 

It's surprising how consistent/predictable this flow is.  As a Dask maintainer it has also been thrilling to see how many people out there use this software, and how many of them we can help get over some of the common early humps of devops, and Dask support.

At the beginning of this period we were very busy as individuals.  Now that we have a machine flowing we're very busy as a company.  The amount of pressure is just as high, but we're more able to scale to adapt to it.

## Where Coiled is today

Today Coiled is focused around [cloud.coiled.io](https://cloud.coiled.io/signup), a hosted Dask-as-a-service product that makes it easy for anyone to be successful scaling their Python data science / machine learning workloads. 

‍

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/6348823dd5c17d614f4de5d5_cost.png)

* Most of our feedback comes from individual users
* Most of our customers are small teams
* Most of our revenue comes from a few large enterprise contracts. 

This is great, we've developed a product that large companies want to pay for that benefits individuals as well.  We get to be a successful business and help the world at the same time.

We've also found that many companies benefit from a little bit of advice.  Dask is designed to be easy to use, but running anything at scale generally benefits from access to expertise.  Coiled generally doesn't take on services contracts (although we partner with and highly recommend [Quansight](https://web.archive.org/web/20220619122148/https://www.quansight.com/) for this work).  However, we have found that we're a good fit for both moderate annual contracts ($250k+) and small Q&A only monthly contracts ($2-5k) in order to make Dask perform optimally for companies.  It turns out that Dask is much more powerful if you have access to some of the core maintainers.

## The future of Coiled

We started the company conservatively, hiring only a few engineers and a skeleton support staff.  This has allowed us to safely test a few hypotheses and learn a few lessons:

1. *A lot* of small teams have started using Dask in the last 6-12 months
2. Companies are quite happy to pay for managed Dask-as-a-service
3. We knew a lot more about what we needed to build than we realized

Because of this, we have high confidence that we're on a good path, and so we're more comfortable spending from our war chest (thank you investors).  We're hiring, and planning to expand significantly over the next few months in a few different directions.

### Product

Our Coiled-on-AWS product is, today, the easiest and fastest way to get started with Dask on the Cloud.  Universally users and customers say that this is exactly the user experience that they want. 

However, they also say that they want many more things, including Azure/GCP support, integration with other products, a smoother GPU experience, and more.  Coiled also falls over from time to time when under heavy load.  These are all great features, and we're working on them, but it's also a lot for the current team to handle on our own.  We're stretched thin and need help [so come join us](https://jobs.lever.co/coiled)! 

### Enterprise

Now that we're within some larger companies there are a whole new set of constraints to deal with.  We have this covered with current staff, but the deals in the sales pipeline loom large over us, and we're likely to be quickly overwhelmed.

### Partnerships

Coiled, like Dask, is designed to solve a specific piece of a larger problem.  Dask adds scalability, but relies on other projects like Pandas/Scikit-Learn for computation and UI.  Coiled adds scalability+DevOps but relies on other products like Prefect/Dash/Sagemaker to provide the other components of a data platform.

Many product companies at about our size see the potential here, and are looking to add a scale button to their product.  We're excited about these opportunities in order to see how effectively we can weave together a collection of similarly aligned companies.

### Dask

Originally Coiled didn't devote substantial resources to Dask itself other than having existing maintainer/employees organize and maintain on the side, and as part of paid work with some of the larger clients (thank you NVIDIA and DE Shaw).

However, there is still *a lot* going on in Dask right now.  We need to step up on maintenance, and also drive forward some of the long running design changes that we've wanted for years.  We're happy to say that we have about one maintainer joining the company per month for the next few months.  We're excited about the added bandwidth that this will give the project.

## Final Thoughts

We've built a machine that both generates revenue and serves the open-source community.  Commercial open source businesses are hard to get right, and while we're not there yet, I think that we're headed in a good direction.

In a recent [Talk Python to Me podcast ](https://web.archive.org/web/20220619122148/https://www.youtube.com/watch?v=xV-asqgKaL4)with other startup founders, we talked a little bit about a sailing analogy to commercial open source companies.

![](https://cdn.prod.website-files.com/63192998e5cab906c1b55f6e/634881e46940a606e80a3235_pexels-min-an-1213982.jpeg)

* Working on your own in OSS is like rowing a boat.  You have full control on where you go, but you can only go so fast, and you can't stop rowing
* Building a business around OSS introduces wind.  You go much faster, don't have to row all the time, but you lose control about exactly where you end up
* Our goal is to build directional technology like rudders and jibs, which allow us to harness wind (commercial interest) and direct it to where we want to go (saving the world)

Coiled today looks like a little skiff, with a jury-rigged rudder and a plastic sheet as a front sail.  It works!  But it could also use some love.  We're in calm waters and winds are strong, and so it feels like we could have a large impact. 

Coiled today has grown a lot from me sitting alone in an AirBnB in Oakland, CA.  We're about twenty people strong today.  However, we're also still very young and need scrappy individuals to jump on board and help us build stuff ranging from product, to Dask itself, to messaging, to sales infrastructure and operations.  If you're looking for a new challenge with the potential for high impact then please take a look at our [current jobs postings](https://jobs.lever.co/coiled), or if nothing there looks right, but you think that you yourself would be a good fit then e-mail us at jobs.coiled.io.  At our stage companies are more about high-impact people than specific roles.

‍

‍

<div class="w-embed"><span class="hs-cta-wrapper" id="hs-cta-wrapper-03d656c6-4957-4620-9331-31dd2182c1ec">
  <span class="hs-cta-node hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec" id=""hs-cta-03d656c6-4957-4620-9331-31dd2182c1ec"" data-hs-drop="true" style="visibility: visible;"><a id="cta_button_9245528_be9599a4-6c30-4743-aa64-3e10e4ddc6b6" class="cta_button text-center" href="https://content.coiled.io/cs/c/?cta_guid=be9599a4-6c30-4743-aa64-3e10e4ddc6b6&amp;signature=AAH58kGdeen7L61UFR9LvyOcFaG_Lm1yAw&amp;portal_id=9245528&amp;placement_guid=03d656c6-4957-4620-9331-31dd2182c1ec&amp;click=56399723-0285-4223-81a2-0dc2db285b9d&amp;redirect_url=APefjpEgGEjowsRL-tG0p_bfx0EDPmjbQQv_e0ZJAcSNKukmlFxa-BZD9L-QgxFMsAFUChtOAjSwqzIGevTmd31ABZu1ZtJrwGHlN8u1efP9MxwFnZlMGW0&amp;hsutk=&amp;canon=https%3A%2F%2Fwww.