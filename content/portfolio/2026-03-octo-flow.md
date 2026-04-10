+++
title = "Octo-Flow: A High-Performance CLI for Processing Large GitHub Event Datasets"
description = "High-performance Rust CLI for processing large GitHub event datasets"
date = 2026-03-01
authors = ["Iago Bozza"]
draft = false

[taxonomies]
tags = ["portfolio", "rust", "streaming", "json", "cli", "github"]
+++

* **GitHub:** [https://github.com/writeonlycode/octo-flow](https://github.com/writeonlycode/octo-flow)
* **Crates.io:** [https://crates.io/crates/octo-flow](https://crates.io/crates/octo-flow)

**Octo Flow** is a high-performance command-line tool written in Rust for
processing large streams of GitHub event data.

<!-- more -->

## The Problem

GitHub Archive datasets are published as **compressed NDJSON files**, often
reaching gigabytes in size. Processing this data efficiently is challenging:

* Loading the full dataset into memory is not feasible
* Traditional tools can be slow or memory-intensive
* Simple tools like `grep` lack structured JSON parsing

## The Solution

Octo Flow processes NDJSON streams **line-by-line**, transforming raw event
data into clean tabular output while maintaining a **constant memory footprint**.

It is designed for:

* data pipelines
* log processing
* analytics workflows
* ETL preprocessing

## Example

```bash
curl https://data.gharchive.org/2026-03-11-15.json.gz \
| zcat \
| octo-flow --input - --event WatchEvent
```

### Example Output

```bash
2489651057	2015-01-01T15:00:03Z	SametSisartenep	visionmedia/debug	WatchEvent
2489651078	2015-01-01T15:00:05Z	comcxx11	phpsysinfo/phpsysinfo	WatchEvent
2489651080	2015-01-01T15:00:05Z	Soufien	wasabeef/awesome-android-libraries	WatchEvent
```

## Architecture

The tool uses a streaming pipeline:

```
input stream
    ↓
BufReader
    ↓
line iterator
    ↓
serde_json parser
    ↓
event filter
    ↓
TSV output
```

This design allows **multi-gigabyte datasets to be processed using only a few
megabytes of memory**.

## Technical Highlights

* Streaming JSON parsing with Serde
* Zero-copy deserialization using `&str`
* Buffered I/O with `BufReader`
* Iterator-based processing
* Structured error handling with `thiserror`
* CLI integration tests using `assert_cmd`

## Performance

Benchmark on a ~9.5MB dataset (~65k events):

| Tool      | Time   |
| --------- | ------ |
| jq        | 0.40s  |
| octo-flow | 0.053s |
| grep      | 0.001s |

While `grep` is faster, it does not perform structured parsing.
Octo Flow achieves **near-native performance with full JSON awareness**.

## Outcome

Octo Flow demonstrates how to build **efficient, streaming data pipelines in
Rust**, combining:

* predictable memory usage
* high throughput
* strong type safety

This approach is especially useful for systems that need to process large
volumes of data reliably.
