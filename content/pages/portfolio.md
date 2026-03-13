+++
title = "Portfolio"
path = "portfolio"
+++

I build reliable software with a focus on **performance, simplicity, and
maintainability**. My work spans **modern web applications** and **systems
programming in Rust**.

---

## Octo Flow

- **GitHub:** [https://github.com/writeonlycode/octo-flow](https://github.com/writeonlycode/octo-flow)
- **Crates.io:** [https://crates.io/crates/octo-flow](https://crates.io/crates/octo-flow)

**Octo Flow** is a high-performance command-line tool written in Rust for
processing large streams of GitHub event data.

It reads **newline-delimited JSON (NDJSON)** event streams and converts them
into clean tabular reports while maintaining a **constant memory footprint**.
The tool was designed for **data pipelines, log processing, and analytics
workflows** where large JSON datasets must be processed efficiently.

### Key Concepts

- **Streaming architecture**: processes JSON line-by-line rather than loading
entire datasets into memory.

- **Zero-copy deserialization**: event fields are parsed using references
instead of allocating new strings.

- **Unix pipeline friendly**: integrates naturally with shell tools like
`curl`, `zcat`, and pipes.

- **Structured error handling:** implemented with `thiserror`, using a custom
error enum to represent different failure cases.

### Example

```bash
curl https://data.gharchive.org/2026-03-11-15.json.gz
| zcat
| octo-flow --input - --event WatchEvent > stars.tsv
```

### Streaming Pipeline Architecture

```bash
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

This streaming design allows **multi-gigabyte datasets to be processed while
using only a few megabytes of memory**.

### Technical Highlights

- Streaming JSON parsing using **Serde**
- Buffered I/O with `BufReader`
- Iterator-based data processing
- End-to-end CLI testing using `assert_cmd`
- Installable via `cargo install`

