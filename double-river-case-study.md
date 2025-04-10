---
title: Double River: Enhanced Algorithmic Trading Performance
description: Double River, a quant trading fund specializing in algorithmic trading, was able to use Coiled to optimize their compute-heavy processes like signal generation and backtesting.
blogpost: true
date: 
author: 
---

# Double River: Enhanced Algorithmic Trading Performance

## About Double River Investments

Double River Investments is a quant trading fund operating across global markets. With a commitment to both financial success and positive global impact, the company leverages data-driven strategies to maximize returns while also dedicating resources to making a difference. In addition to its focus on global markets, Double River Investments actively engages in impact and humanitarian investments in developing countries, aiming to improve local communities and contribute to sustainable growth where it's needed most. This dual focus on profit and purpose underscores the firm's broader mission to create long-term value beyond the financial markets.

Double River specializes in algorithmic trading. They employ sophisticated data pipelines to process vast amounts of financial data, generate signals, and execute trading strategies. Their primary challenge is optimizing compute-heavy processes like signal generation and backtesting, which require significant memory and computational power.

## Challenges at scale

Before partnering with Coiled, Double River faced several challenges:

1. **Hardware limitations**: Using Google Cloud's Cloud Run limited Double River to jobs requiring no more than 32GB of RAM. This worked fine for smaller workflows, but for larger jobs this forced Nelson and his team to spin up larger virtual machines (VMs) manually.
2. **Backtesting bottlenecks**: Backtesting strategies—running simulations over historical data to assess the viability of trading strategies—was a critical but slow process. These tests could only be run sequentially due to dependencies between each day's trading data, significantly slowing down decision-making.‍
3. **Infrastructure management overhead**: Manual VM management for backtests was time-consuming and inefficient, often leading to costly errors like forgotten active VMs. 

## Worry-free cloud infrastructure with Coiled

Coiled's solution addressed Double River's infrastructure and compute challenges, providing a seamless integration with their existing cloud environment while significantly reducing manual management. Key benefits included:

1. **Hardware flexibility**: With Coiled, Double River can now dynamically scale compute resources based on job requirements. Coiled enabled the team to spin up lightweight Cloud Run jobs to launch larger, memory-optimized Coiled environments without needing to manually manage VMs.
2. **Increased backtesting capacity**: Coiled's ability to run parallel workloads allowed Double River to significantly increase the number of backtests they could execute simultaneously—from five to as many as 19—removing backtesting as a bottleneck to their workflow.‍
3. **Reduced infrastructure management**: Researchers could now spin up Coiled environments directly, eliminating the need for engineers to manage and provision VMs, reducing operational overhead and improving workflow efficiency.

## Results

> "With Coiled, we've dramatically increased our backtesting throughput and removed the bottlenecks from our decision-making process. We no longer worry about managing VMs, and our researchers can focus on what they do best—building and testing better trading strategies."

1. **Faster decision-making:** With Coiled, Double River was able to test new trading strategies and algorithms faster by running multiple backtests simultaneously. This resulted in faster decision-making when determining whether to implement new features or signals.
2. **Cost efficiency**: Coiled helped eliminate "dead costs" like forgotten VMs. Approximately 30% of their cloud spend was once wasted on non-productive expenses, now reduced to zero. They have also been able to [easily spin up ARM instances](https://docs.coiled.io/blog/dask-graviton.html), which are more efficient and often cheaper than the Intel equivalent.
3. **Team acceleration**: Researchers gained autonomy by launching their own environments, significantly improving productivity and freeing up the engineering team to focus on more impactful tasks. Coiled automated much of Double River's workflow, enabling researchers to spin up machines easily without needing to manage infrastructure. This self-service model freed up engineering resources, saving about 10% of the engineering team's time that was previously spent on manual VM management.