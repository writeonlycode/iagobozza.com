+++
title = "Learning Vectors in Rust by Solving 10 Small Problems"
description = "Learn Rust Vec through 10 practical exercises. Master push(), pop(), get(), transform (map), filter, mutate in place, remove with clear examples."
date = 2026-03-31
authors = ["Iago Bozza"]
draft = true

[taxonomies]
tags = ["rust", "vectors"]
+++

Vectors show up everywhere in real Rust code: collecting results, transforming
data, buffering streams, and serving as the backbone of iterator pipelines.
They're also where ownership, borrowing, and iteration patterns stop being
abstract ideas and start becoming practical tools you use constantly.

This post follows the same approach as the previous one: 10 progressively more
complex exercises, each focused on a single idea, written in idiomatic Rust,
and designed to highlight real decisions around ownership and borrowing.

<!-- more -->

---

## Prerequisites

You should be comfortable with basic Rust syntax, `&str`, `String`, ownership
vs borrowing, and basic iteration. We'll use only the standard library.

---

## 1. Push & Grow (The Foundation of Vec)

Goal: Learn how a `Vec` is built dynamically using `Vec::new` and `push`.

```rust
pub fn build_vec(n: i32) -> Vec<i32> {
    let mut v = Vec::new();

    for i in 1..=n {
        v.push(i);
    }

    v
}
```


This is the simplest meaningful problem you can solve with a vector: building
one from scratch. You start with an empty `Vec` and gradually add elements to
it. That “start empty, then push” pattern shows up everywhere in real Rust
code, especially when transforming or collecting data.

If there’s one idea you need to internalize early about vectors in Rust, it’s
this:

> A Vec<T> is a growable, contiguous array.

That word growable is doing a lot of work.  Every time you call `push`, the
vector may need to grow its internal buffer. When that happens, it allocates a
larger chunk of memory and moves the existing elements. But Rust does this in a
way that minimizes how often reallocations happen, so `push` is *amortized
O(1)*.

The important mental model here is that a `Vec` is not just a list. It’s a
**managed buffer**. It keeps track of its length and capacity, growing as
needed while keeping elements stored contiguously in memory.

You'll use this exact pattern (initialize, loop, push) whenever you need to build
a collection step by step. Once this feels natural, working with vectors
becomes much more intuitive.

---

## 2. Safe Access (Indexing Without Panic)

Goal: Learn the difference between direct indexing and safe access with `.get()`.

```rust
pub fn safe_get(v: &[i32], idx: usize) -> Option<i32> {
    v.get(idx).copied()
}
```

Accessing elements in a vector seems simple, but Rust makes you confront an
important reality: **what if the index doesn’t exist?**

You *can* write `v[idx]`, but that will panic if `idx` is out of bounds. In
many languages, that kind of failure is either undefined or silently wrong. In
Rust, it’s explicit and avoidable.

That’s where `.get()` comes in. Instead of assuming the index is valid, it
returns an `Option<&T>`. This forces you to handle both cases:

* `Some(&value)` if the index exists
* `None` if it doesn’t

In the example, we call `.copied()` to turn an `Option<&i32>` into an
`Option<i32>`, since `i32` is a `Copy` type. This keeps the function ergonomic
while still being completely safe.

The deeper lesson is about **embracing uncertainty in data access**. Any time
you’re working with indices, especially in real systems where data might be
incomplete or dynamic, `.get()` is the tool that keeps your code robust.

You’ll see this pattern often: prefer safe access when there’s any doubt. It’s
one of the small habits that makes Rust code feel solid instead of fragile.

---

## 3. Iteration Basics (Reading Without Taking Ownership)

Goal: Learn how to iterate over a vector using references.

```rust
pub fn sum(v: &[i32]) -> i32 {
    let mut total = 0;

    for x in v {
        total += *x;
    }

    total
}
```

At this point, you know how to build a `Vec` and how to access individual
elements safely. The next step is learning how to **process all
elements**, which is where iteration comes in.

The key detail in this kata is the function signature: `&[i32]`. Instead of
taking ownership of a `Vec<i32>`, we take a **slice**. This makes the function
more flexible: it can accept both vectors and arrays, and it avoids unnecessary
allocations or moves.

Inside the loop:

```rust
for x in v
```

`x` is not an `i32`, but a `&i32`. You’re iterating over **references**, not
values. That’s why you need to explicitly dereference it with `*x` when adding
to the total.

This is one of the first places where Rust’s ownership model becomes tangible:

* You’re **borrowing** the data, not consuming it
* The original vector remains usable after the function call
* Mutation is not allowed (because you only have `&`, not `&mut`)

This pattern: looping over references, is extremely common in Rust. It lets you
read data efficiently without taking ownership or cloning.

Once this clicks, you stop thinking “loop over a vector” and start thinking:

> “Am I borrowing, mutating, or consuming this data?”

---

## 4. Mapping (Building a New Vector)

Goal: Learn how to transform a vector into another by building a new `Vec`.

```rust
pub fn square_all(v: &[i32]) -> Vec<i32> {
    let mut result = Vec::new();

    for x in v {
        result.push(x * x);
    }

    result
}
```

This is your first real **data transformation** problem: take some input, and
produce a new collection with modified values.

The important shift here is that you’re no longer just reading data—you’re
**creating a new vector based on existing data**.

Just like in the exercise, you start with an empty `Vec`. But now, instead of
pushing arbitrary values, you push **derived values**:

```rust
result.push(x * x);
```

Notice something subtle: `x` is still a `&i32`, but Rust allows you to use it
in arithmetic like this because it automatically dereferences when needed. You
could write `(*x) * (*x)`, but that quickly becomes noisy, so Rust helps you out
here.

This pattern:

* create an empty `Vec`
* iterate over input
* push transformed values

is one of the most important patterns in real-world Rust. It shows up anytime
you’re reshaping data, whether that’s parsing, filtering, or preparing results.

You might recognize this as something other languages solve with `map`. Rust
has that too, but learning to do it manually first builds a much stronger
mental model of what’s actually happening.

---

## 5. Filtering (Selective Copying)

Goal: Learn how to build a new vector by **selectively keeping elements**.

```rust
pub fn evens(v: &[i32]) -> Vec<i32> {
    let mut result = Vec::new();

    for x in v {
        if x % 2 == 0 {
            result.push(*x);
        }
    }

    result
}
```

This exercise introduces a small but powerful twist on the previous one:
instead of transforming every element, you now **decide which elements
survive**.

The structure is almost identical:

* start with an empty `Vec`
* iterate over the input
* push into a result vector

But now there’s a gate:

```rust
if x % 2 == 0
```

Only values that satisfy the condition make it into the output.

This is the essence of **filtering**, a pattern you’ll use constantly when
dealing with real data: removing invalid entries, extracting relevant records,
or narrowing down results.

There’s also a small but important detail here. Since `x` is a `&i32`, you
explicitly dereference it when pushing:

```rust
result.push(*x);
```

Unlike arithmetic, Rust won’t automatically dereference in this context, so you
stay in control of what’s being copied.

The deeper lesson is that you’re now combining:

* **iteration** (from the previous kata)
* **conditionals** (`if`)
* **vector construction** (`push`)

into a single pattern:

> Iterate → decide → keep (or skip)

---

6. In-Place Mutation (Changing Data Without Reallocating)

Goal: Learn how to modify a vector **in place** using mutable references.

```rust
pub fn double_in_place(v: &mut Vec<i32>) {
    for x in v.iter_mut() {
        *x *= 2;
    }
}
```

So far, every transformation you’ve done created a **new vector**. That’s often
the right choice, but sometimes, you don’t want a new allocation. You just want
to **change what’s already there**.

That’s what this kata is about.

The function takes a `&mut Vec<i32>`, which means:

* You are borrowing the vector
* You are allowed to **mutate it**
* The caller keeps ownership

Inside the loop:

```rust
for x in v.iter_mut()
```

You’re no longer iterating over `&i32`, but over `&mut i32`. That’s a big
shift. Each `x` is a **mutable reference to an element inside the vector**.

That’s why mutation looks like this:

```rust
*x *= 2;
```

You explicitly dereference the mutable reference and update the value in place.

This pattern is extremely important in performance-sensitive code. Instead of
allocating a new vector and copying data over, you reuse the existing memory
and just modify the contents.

The deeper lesson here is about **control over mutation**:

* `&[T]` → read-only access
* `&mut Vec<T>` → full control to change the data

Rust forces you to be explicit about this distinction, and that’s what keeps
mutation safe and predictable.

Once this clicks, you start thinking:

> “Do I need a new vector, or can I transform this one in place?”

---

## 7. Removing Elements (Dealing with Vec’s Weak Spot)

Goal: Learn how to remove elements manually and understand the cost of doing
so.

```rust
pub fn remove_all(v: &mut Vec<i32>, target: i32) {
    let mut i = 0;

    while i < v.len() {
        if v[i] == target {
            v.remove(i);
        } else {
            i += 1;
        }
    }
}
```

Up to now, `Vec` has felt smooth and efficient, but this kata exposes one of
its key trade-offs:

> Removing elements from the middle is **expensive**.

Why? Because a `Vec` stores elements contiguously in memory. When you remove an
element, everything after it has to **shift left** to fill the gap.

This implementation makes that behavior explicit.

We use a `while` loop instead of a `for` loop because we need **full control
over the index**:

* If we remove an element, we **don’t increment `i`**, because a new value has shifted into that position
* If we keep the element, we move forward as usual

```rust
v.remove(i);
```

That single line hides a potentially costly operation: all subsequent elements
are moved.

This kata is less about writing the perfect solution and more about
understanding the **data structure’s limitations**.

In real-world code, you might prefer alternatives:

* Build a new filtered vector (like in the previous kata)
* Use `.retain()` for a more idiomatic approach

But doing it manually once is important. It gives you a concrete mental model:

> A `Vec` is fast at the end, but costly in the middle.

---

## 8. Stack Behavior (Using Vec as LIFO)

Goal: Learn how to use a `Vec` as a **stack** with `push` and `pop`.

```rust
pub fn validate_parentheses(s: &str) -> bool {
    let mut stack = Vec::new();

    for ch in s.chars() {
        if ch == '(' {
            stack.push(ch);
        } else if ch == ')' {
            if stack.pop().is_none() {
                return false;
            }
        }
    }

    stack.is_empty()
}
```

So far, you’ve been thinking of a `Vec` as a collection, but one of its most
important real-world uses is as a **stack**.

A stack follows **Last-In, First-Out (LIFO)**:

* `push` → add to the top
* `pop` → remove from the top

This makes `Vec` perfect for problems involving **nesting**, **matching**, or
**undo-like behavior**.

In this kata, you validate whether parentheses are balanced. The idea is
simple:

* Every `'('` gets pushed onto the stack
* Every `')'` tries to pop from the stack

If you try to pop and the stack is empty, something went wrong—there’s no
matching `'('`.

```rust
if stack.pop().is_none()
```

That’s your failure case.

At the end, the stack must also be empty. If it’s not, there are unmatched
`'('` left over.

This pattern shows up everywhere:

* parsing expressions
* validating syntax
* traversing nested structures

And the key insight is this:

> A `Vec` is not just a list. It’s a **general-purpose tool** that can act as a
> stack with zero extra cost.

---

## 9. Capacity & Performance (Thinking Beyond Length)

Goal: Learn how to **avoid unnecessary reallocations** using `with_capacity`.

```rust
pub fn build_with_capacity(n: usize) -> Vec<i32> {
    let mut v = Vec::with_capacity(n);

    for i in 0..n {
        v.push(i as i32);
    }

    v
}
```

Up until now, you’ve been building vectors with `Vec::new()`. That works
perfectly, but it hides an important detail:

> A `Vec` has both a **length** and a **capacity**.

* **Length** is how many elements are currently stored
* **Capacity** is how many elements it can hold *without reallocating*

When you keep calling `push`, the vector occasionally runs out of space and
needs to grow its internal buffer. That means allocating new memory and copying
all elements over.

This is efficient in an amortized sense, but if you already know how many
elements you’ll need, you can do better.

That’s where this comes in:

```rust id="q1z7vb"
Vec::with_capacity(n)
```

You’re telling Rust:

> “Allocate enough space for `n` elements upfront.”

Now, all pushes happen without reallocations.

This pattern matters in performance-sensitive code, especially when:

* building large collections
* running tight loops
* working in systems where allocations are costly

The deeper lesson is that you’re starting to think not just about *what* your
code does, but *how it behaves in memory*.

Most of the time, `Vec::new()` is fine. But when performance matters,
`with_capacity` gives you a simple and powerful optimization.

---

## 10. Partitioning Data (Splitting into Multiple Outputs)

Goal: Learn how to route elements into **multiple vectors** based on a
condition.

```rust
pub fn partition(v: &[i32]) -> (Vec<i32>, Vec<i32>) {
    let mut negatives = Vec::new();
    let mut positives = Vec::new();

    for x in v {
        if *x < 0 {
            negatives.push(*x);
        } else {
            positives.push(*x);
        }
    }

    (negatives, positives)
}
```

This final exercise ties everything together: iteration, conditionals, and
building vectors—but now with a twist:

> You’re not producing one result. You’re producing **multiple outputs at the
> same time**.

The structure is familiar:

* iterate over the input
* decide what to do with each element
* push into a vector

But instead of a single destination, you now **route data**:

```rust id="c3m8hz"
if *x < 0 {
    negatives.push(*x);
} else {
    positives.push(*x);
}
```

Each element is classified and sent to the appropriate bucket.

This pattern shows up constantly in real-world systems:

* separating valid vs invalid data
* grouping items by category
* routing events in pipelines

And notice the return type:

```rust
(Vec<i32>, Vec<i32>)
```

Rust makes it easy to return multiple values, so you can model this kind of
split cleanly without extra structs or boilerplate.

The deeper lesson here is that you’re no longer just transforming
collections. You’re **organizing data flows**.

## Final Thoughts

At this point, you’ve seen the core patterns:

* build (`push`)
* read (`&[T]`)
* transform (map)
* filter
* mutate in place
* remove
* use as a stack
* optimize capacity
* split into outputs

That’s the foundation of thinking with `Vec`.

Once these patterns feel natural, vectors stop being just a data structure, and
start becoming a **tool you can shape to fit almost any problem**.

