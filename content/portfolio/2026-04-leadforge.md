+++
title = "LeadForge: A High-Performance CLI for Lead Generation from Hacker News"
description = "High-performance Rust CLI for concurrent lead generation from real-world data sources."
date = 2026-04-10
authors = ["Iago Bozza"]
draft = false

[taxonomies]
tags = ["portfolio", "rust", "cli", "async", "scraper", "hacker-news", "lead-generation" ]
+++

* **GitHub:** [https://github.com/writeonlycode/leadforge](https://github.com/writeonlycode/leadforge)
* **Crates.io:** [https://crates.io/crates/leadforge](https://crates.io/crates/leadforge)

LeadForge is a high-performance Rust CLI for concurrent lead generation from
real-world data sources. It fetches and processes job listings (currently from
Hacker News) using an async streaming pipeline with bounded concurrency, retry
logic, and filtering. It's designed for building real lead generation
workflows.

<!-- more -->

## The Problem

I set out to build a simple idea: a CLI tool that could reliably extract
**high-quality job leads** from real-world data sources. Hacker News seemed
like a perfect starting point. It has a public API, a steady stream of job
postings, and a developer-focused audience.

But there was a catch: The API doesn't provide a clean, complete "jobs feed."
Instead, it exposes two endpoints:

* `/jobstories`: a **curated list** of job postings
* `/maxitem`: the **latest item ID** in a massive global dataset

Each endpoint has a different failure mode:

The `jobstories` endpoint is high signal, but low coverage: it contains only job posts,
but it returns a **limited number of items** that is not enough for real lead
generation workflows.

Starting with the `maxitem` endpoint, we can get high coverage, but it's
extremely noisy: if we scan the IDs backwards, it gives access to *everything*
(stories, comments, polls, jobs…), and most items are **not jobs**.

So the challenge becomes:

> How do you get *enough* results without drowning in irrelevant data?

## The Naive Approach (and Why It Fails)

The obvious solution is:

> Start from `maxitem` and walk backwards until you find enough jobs.

This works, but poorly: we end up wasting a massive number of requests, and most
responses are irrelevant. So, we have a high latency before finding useful
results and a lot of wasted bandwidth.

In other words:

> **Terrible signal-to-noise ratio**

## This Is a Signal vs Recall Problem

This isn't just an API issue. It's a classic tradeoff:

* **Signal (precision)**: How relevant your results are
* **Recall (coverage)**: How many results you can find

The endpoints map perfectly to this:

| Source       | Signal | Recall |
| ------------ | ------ | ------ |
| jobstories   | High   | Low    |
| maxitem scan | Low    | High   |

So the solution isn't to pick one. It’s to **combine both**.

## The Solution: A Hybrid Ingestion Strategy

LeadForge implements a **hybrid pipeline**:

1. Start with `jobstories` to get high-quality leads first
2. Fall back to `maxitem` walkback only if more results are needed
3. Merge both into a **single descending stream of IDs**

Conceptually:

* curated jobs (high signal)
* recent global items (high coverage)
* single unified stream

In practice, this looks like:

* Fetch both sources concurrently
* Sort and combine IDs
* Traverse from newest to oldest
* Process everything through the same pipeline

This approach solves both sides of the problem: because the curated job IDs are
included in the stream, the pipeline naturally yields **relevant jobs early**.
If the curated list isn't enough, the system automatically expands into the
broader dataset.

The key detail is early termination: the pipeline stops as soon as it collects
`limit` results. It doesn't scan the entire dataset. So in most cases you get
results from `jobstories` and you *barely touch* the noisy fallback.

## Implementation Highlights

This strategy is powered by a streaming, concurrent pipeline. Data is processed
as it arrives, so:

* No large intermediate collections
* Lower memory usage
* Faster time-to-first-result

Requests are executed concurrently, but within limits:

* High throughput
* No resource exhaustion
* No API overload

Transient failures are handled automatically:

* timeouts
* connection issues
* 5xx errors
* rate limits

Each request is retried with exponential backoff and jitter. Only relevant data
flows forward:

* Non-job items are discarded immediately
* Keyword filtering happens inline
* No extra processing pass

## Closing Thoughts

This project started as a simple CLI. It turned into an exercise in:

* working around imperfect APIs
* designing for signal over noise
* building pipelines that reflect real-world constraints

The result is a system that doesn't just fetch data. It **prioritizes what
matters**.

