+++
title = "Learning HashMaps in Rust by Solving 10 Small Problems"
description = "Learn Rust HashMaps through 10 practical exercises. Master entry(), ownership, grouping, merging, filtering, and nested maps with clear examples."
date = 2026-02-27
authors = ["Iago Bozza"]

[taxonomies]
tags = ["rust", "hashmaps"]
+++

HashMaps are one of the most common data structures in Rust, and also one of
the first places where ownership, borrowing, and references really start to
matter.

This post walks through 10 progressively more complex exercises that cover
almost every practical HashMap pattern you'll use in real Rust code. Each
exercise focuses on one idea, uses idiomatic Rust, highlights ownership and
borrowing decisions, and builds on the previous ones.

If you can write and understand all of these, you're no longer "learning
HashMap". You're using it fluently.

<!-- more -->

---

## Prerequisites

You should be comfortable with basic Rust syntax, `Vec`, `&str`, `String`,
ownership vs borrowing, and iterators and `for` loops.

We’ll use only the standard library:

```rust
use std::collections::HashMap;
```

---

## 1. Counting words (the canonical HashMap problem)

**Goal:** Learn the `entry()` + `or_insert()` pattern.

```rust
pub fn word_count(words: &[&str]) -> HashMap<String, usize> {
    let mut map = HashMap::new();

    for word in words {
        *map.entry(word.to_string()).or_insert(0) += 1;
    }

    map
}
```

This is usually the first real problem people solve with a hash map, and in
Rust it introduces the most important part of the API: `entry()`. Instead of
checking whether a key exists and then updating or inserting manually,
`entry()` gives you a handle that represents *either* case. Calling
`or_insert(0)` ensures a value is present and returns a mutable reference to
it.

That return type is the key detail: `or_insert` yields a `&mut usize`, not a
value. Rust requires you to explicitly dereference it before mutating, which is
why the `*` is necessary. This may feel a bit verbose at first, but it’s Rust
making mutation explicit and safe.

You’ll see this exact pattern -- `*map.entry(key).or_insert(0) += 1` -- over
and over again in real Rust code, from frequency counters to aggregations. Once
it clicks, working with `HashMap` stops feeling awkward and starts feeling
precise and powerful.

---

## 2. Character frequency

**Goal:** Same idea, different key type.

```rust
pub fn char_frequency(s: &str) -> HashMap<char, usize> {
    let mut map = HashMap::new();

    for c in s.chars() {
        *map.entry(c).or_insert(0) += 1;
    }

    map
}
```

This example reinforces that the `entry()` pattern is completely independent of
the key type. Instead of owned `String`s, we’re now using `char` keys produced
by iterating over the string. Because `char` is a small, `Copy` type, we can
insert it directly into the map without any allocation or cloning.

What’s important here is that nothing about the counting logic changes. Once
you understand how `entry()` and `or_insert()` work, switching from words to
characters—or to any other key type—is mostly mechanical. This is a recurring
theme in Rust: the hard part is learning the pattern; applying it to different
data is easy.

The repetition is intentional. By solving the same problem with a slightly
different shape, you start to see `HashMap` as a flexible tool rather than a
special-case container.

---

## 3. Safe lookup with a default value

**Goal:** Read from a `HashMap` without panicking.

```rust
pub fn get_score(scores: &HashMap<String, i32>, name: &str) -> i32 {
    *scores.get(name).unwrap_or(&0)
}
```

This example shifts focus from insertion to safe reading. When you call `get`,
Rust returns an `Option<&i32>`, not the value itself. That’s deliberate: the
key might not exist, and Rust forces you to acknowledge that possibility.
Instead of unwrapping and risking a panic, we provide a fallback with
`unwrap_or(&0)`.

Notice that the fallback is a reference to `0`, not just `0`. That’s because
`get` returns a reference, so the types must match. After that, we dereference
the result with `*` to obtain the actual `i32`. Since `i32` implements `Copy`,
dereferencing simply copies the value out—no ownership complications, no moves.

The pattern here is subtle but important: you borrow from the map, supply a
safe default, and only then copy the value if needed. It’s a compact example of
how Rust combines `Option`, references, and the type system to eliminate whole
classes of runtime errors while keeping the code concise and expressive.

---

## 4. Merging two HashMaps

**Goal:** Combine values with the same key.

```rust
pub fn merge_counts( a: HashMap<String, usize>, b: HashMap<String, usize>,) -> HashMap<String, usize> {
    let mut map = a;

    for (key, value) in b {
        *map.entry(key).or_insert(0) += value;
    }

    map
}
```

This function builds directly on the counting pattern, but now applied across
two maps. By taking ownership of both `a` and `b`, it avoids any need for
cloning: `a` becomes the base map, and `b` is consumed as we iterate over it.
Each key–value pair from `b` is moved into the result, and if the key already
exists, its value is incremented rather than replaced.

The important detail is that `entry` works just as well when merging as it does
when counting from scratch. Whether a key is new or already present, the logic
stays the same, which makes this pattern easy to reuse and reason about. The
result is code that is both efficient—because it moves data instead of copying
it. And expressive, because the intent (“add these counts together”) is
immediately clear.

---

## 5. Finding the most frequent entry

**Goal:** Iterate over entries and return a derived value.

```rust
pub fn most_frequent(map: &HashMap<String, usize>) -> Option<String> {
    map.iter()
        .max_by_key(|(_, count)| *count)
        .map(|(key, _)| key.clone())
}
```

This example moves from mutation to analysis. Instead of inserting or merging,
we iterate over the existing entries and compute a derived result. Calling
`iter()` yields `(&K, &V)`, meaning we’re borrowing both the keys and the
values. Nothing is moved out of the map, and the function remains read-only.

`max_by_key` is where the intent becomes explicit. Rather than manually
tracking the “largest so far,” we tell Rust exactly what we want: select the
entry with the maximum `count`. The closure receives references, so we
dereference `count` to compare its numeric value.

Finally, we convert from `Option<(&String, &usize)>` to `Option<String>` using
`map`. The only allocation happens at the boundary, where we clone the key for
the return value. Everything else stays borrowed. This is a common Rust
pattern: iterate by reference for efficiency, then clone only when you must
hand ownership to the caller.

The result is compact, expressive, and avoids unnecessary work. Very much in
line with idiomatic Rust design.

---

## 6. Grouping values (HashMap of Vecs)

**Goal:** Store multiple values per key.

```rust
pub fn group_by_first_letter(words: &[&str]) -> HashMap<char, Vec<String>> {
    let mut map = HashMap::new();

    for word in words {
        if let Some(first_char) = word.chars().next() {
            map.entry(first_char)
                .or_insert_with(Vec::new)
                .push(word.to_string());
        }
    }

    map
}
```

This exercise introduces a very common variation of the `entry` pattern:
storing *collections* as values. Instead of counting, each key maps to a `Vec`
that accumulates multiple items. Conceptually, this is how you model "group by"
operations in Rust.

The key detail is the use of `or_insert_with(Vec::new)`. Unlike
`or_insert(Vec::new())`, which would eagerly allocate a vector even when the
key already exists, `or_insert_with` only constructs the `Vec` when it's
actually needed. That keeps the code efficient while remaining clear and
expressive.

Once the entry exists, `push` works exactly as you'd expect: you get mutable
access to the vector and append a new value. This pattern shows up
everywhere—grouping records, building indices, or collecting results by
category—and mastering it is a big step toward writing fluent, idiomatic Rust.

---

## 7. Inverting a HashMap

**Goal:** Swap keys and values.

```rust
pub fn invert_map(map: HashMap<String, i32>) -> HashMap<i32, String> {
    let mut inverted = HashMap::new();

    for (key, value) in map {
        inverted.insert(value, key);
    }

    inverted
}
```

This example is all about ownership and intent. By taking the input `HashMap`
by value, the function is free to *consume* it, moving each key–value pair out
without cloning. That makes the transformation both simple and efficient: we
iterate over the original map, swap the positions of `key` and `value`, and
insert them into a new one.

The important subtlety is semantic rather than technical. A `HashMap` can only
have one value per key, so if the original map contains duplicate values, later
inserts will overwrite earlier ones. Rust doesn't prevent this because it's not
a correctness issue. It's a modeling choice. In some cases this behavior is
exactly what you want; in others, you might prefer a `HashMap<i32,
Vec<String>>` to preserve all associations.

This exercise highlights a recurring theme in Rust: the type you choose encodes
the guarantees you get. Here, the choice of `HashMap<i32, String>` explicitly
says “one key wins,” and the code cleanly enforces that decision.

---

## 8. Filtering entries

**Goal:** Build a new `HashMap` from a predicate.

```rust
pub fn filter_above(map: &HashMap<String, i32>, threshold: i32) -> HashMap<String, i32> {
    let mut filtered = HashMap::new();

    for (key, value) in map {
        if *value > threshold {
            filtered.insert(key.clone(), *value);
        }
    }

    filtered
}
```

This exercise focuses on selective ownership. The input map is borrowed, which
means we’re not allowed to move anything out of it. Instead, we iterate over
references and decide which entries are worth keeping based on a simple
predicate.

When a value passes the threshold check, we explicitly clone the key and copy
the value into the new map. That makes the ownership boundary very clear: only
the entries that survive the filter are duplicated, and everything else remains
borrowed. Since `i32` implements `Copy`, dereferencing `value` is cheap and
natural.

While this could be written using iterator chains like `filter` and `collect`,
the explicit loop keeps the logic readable and approachable. In Rust, clarity
often beats cleverness. Especially when ownership and borrowing are part of the
lesson.

---

## 9. Counting with ownership

**Goal:** Consume a collection.

```rust
pub fn count_numbers(nums: Vec<i32>) -> HashMap<i32, usize> {
    let mut count = HashMap::new();

    for n in nums {
        *count.entry(n).or_insert(0) += 1;
    }

    count
}
```

This example looks almost identical to earlier counting patterns, but the key
difference is in the function signature. By taking `Vec<i32>` by value, the
function consumes the collection. That small choice removes an entire layer of
complexity: there’s no borrowing, no references to juggle, and no need to clone
anything.

Each number is moved directly out of the vector and into the `HashMap` as a
key. Since `i32` implements `Copy`, even that movement is trivial. The `entry`
pattern works exactly as before, but now the ownership model is as simple as it
can possibly be.

This illustrates a powerful Rust principle: if your function logically owns the
data, accept ownership in the type signature. Doing so often eliminates
lifetime concerns and reduces mental overhead. When you can take ownership
cleanly, the code tends to become both simpler and more robust.

---

## 10. Nested HashMaps (real-world aggregation)

**Goal:** HashMap of HashMaps.

```rust
pub fn tally_scores(entries: Vec<(&str, &str, i32)>) -> HashMap<String, HashMap<String, i32>> {
    let mut tally = HashMap::new();

    for (player, game, score) in entries {
        *tally
            .entry(player.to_string())
            .or_insert_with(HashMap::new)
            .entry(game.to_string())
            .or_insert(0) += score;
    }

    tally
}
```

This final exercise combines everything introduced so far into a pattern that
closely resembles real production code. We're aggregating data across two
dimensions: players and games, using a nested `HashMap`. Each outer key maps to
another `HashMap`, which in turn accumulates numeric values.

The logic flows from the outside in. First, we ensure that a map exists for a
given player, creating one only when necessary with
`or_insert_with(HashMap::new)`. From there, we immediately operate on the inner
map, applying the familiar counting pattern to the game and score. Although the
chaining looks dense at first glance, each step follows the same rules you've
already learned.

What's especially important here is how cleanly ownership is handled. The
function consumes the input entries, allocates owned `String`s at the boundary,
and mutates only the structures it owns. There are no lifetimes to track and no
intermediate clones beyond what’s required for storage.

This nested `entry` pattern is common in analytics, logging, metrics, and any
kind of aggregation pipeline.

Once you're comfortable reading and writing code like this, `HashMap`s stop
feeling like a special case and start feeling like a natural extension of
Rust's type system and ownership model!


