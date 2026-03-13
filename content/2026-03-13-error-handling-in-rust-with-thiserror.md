+++
title = "Error Handling in Rust with thiserror"
description = ""
date = 2026-03-13
authors = ["Iago Bozza"]
draft = true

[taxonomies]
tags = ["rust", "thiserror"]
+++

Lorem ipsum.

<!-- more -->

# 1. The Problem with String Errors

Error handling is a central part of Rust’s design. Instead of exceptions, Rust
uses the type system to make failures **explicit and visible in function
signatures**.

Most fallible operations in Rust return the type:

```rust
Result<T, E>
```

This type represents a computation that can either succeed or fail.

* `Ok(T)` — the operation succeeded and produced a value of type `T`
* `Err(E)` — the operation failed with an error value of type `E`

For example:

```rust
fn parse_port(s: &str) -> Result<u16, std::num::ParseIntError> {
    s.parse()
}
```

Here the function signature tells us something very important: parsing may
fail, and if it does, the failure will be represented by a `ParseIntError`.

This design is very different from languages that rely on exceptions. In those
languages, a function may throw errors that are **not visible in the function
signature**, forcing developers to rely on documentation or runtime behavior to
understand what might go wrong.

Rust instead pushes error handling into the type system. When you call a
function that returns a `Result`, the compiler requires you to deal with the
possibility of failure.

This makes error handling **explicit, predictable, and composable**.

## Library Errors vs Application Errors

Rust also tends to distinguish between two kinds of errors:

**Library errors**

These represent failures that library users may want to handle
programmatically. For example:

* invalid input
* missing files
* parsing failures
* network errors

Because callers may want to react differently to these cases, library errors
should be **structured and typed**.

**Application errors**

These usually occur at the boundary of a program, for example in a CLI
application. At this level, errors are often just reported to the user and the
program exits.

Libraries therefore need **well-designed error types**, while applications can
often aggregate and display them.

## The Problem with String Errors

A common beginner pattern is to represent errors as strings:

```rust
fn read_config() -> Result<String, String> {
    Err("config missing".into())
}
```

This compiles, but it creates several problems.

First, the error is **not typed**. A `String` tells us nothing about what kind
of failure occurred. Was the file missing? Was the file unreadable? Was the
format invalid? All of these errors would look the same.

Second, string errors are **impossible to match on** in a reliable way. If
another part of the program wants to handle a specific failure, it would have
to compare strings:

```rust
if err == "config missing" {
    // fragile and brittle
}
```

This approach is brittle and error-prone. A small change in wording can break
the logic.

Third, strings **lose context and structure**. Errors often carry useful
information:

* which file failed to open
* which port number was invalid
* which field failed to parse

Strings flatten all of this information into text, making it difficult to
extract or propagate structured data.

## The Rust Approach

Instead of strings, Rust programs typically define **custom error types**.
These are usually implemented as `enums` that describe the possible failure
modes of an operation.

For example:

```rust
enum ConfigError {
    MissingFile,
    InvalidPort(u16),
}
```

Each variant represents a specific failure, and variants can carry additional
data when necessary.

This approach gives us several advantages:

* errors are **typed**
* callers can **match on specific failures**
* errors can carry **structured data**


# 2. Modeling Errors with Enums

Before introducing `thiserror`, it’s important to understand how error types
are designed in plain Rust. The key idea is simple: **errors are data**.
Instead of representing failures as strings, we model them explicitly using
types.

In Rust, the most common way to represent an error type is with an `enum`.

For example, imagine we are writing a function that reads a configuration file.
Several things could go wrong:

* the file might not exist
* the configuration might contain an invalid port number

We can represent these failures with an enum:

```rust
#[derive(Debug)]
pub enum ConfigError {
    MissingFile,
    InvalidPort(u16),
}
```

Each **variant** of the enum represents a specific failure mode.

* `MissingFile` represents a configuration file that could not be found.
* `InvalidPort(u16)` represents a port number that failed validation, and it carries the invalid value.

This approach has an important advantage: the error type **precisely describes
the possible failures** of the operation.

Instead of returning a vague `String`, our function can return:

```rust
fn read_config() -> Result<String, ConfigError>
```

Now the function signature communicates something meaningful: if the operation
fails, it will fail with a `ConfigError`.

## Enums Are Perfect for Error Modeling

Enums work extremely well for error types because failures usually fall into
**distinct categories**.

Each variant represents a different kind of problem. Variants can also carry
data when additional context is useful.

For example:

* a missing file does not need extra information
* an invalid port benefits from including the problematic value

This makes error handling both **structured and expressive**.

## Matching on Errors

Because the error is a real type, callers can handle different failures
explicitly using pattern matching:

```rust
match err {
    ConfigError::MissingFile => {
        println!("Configuration file not found");
    }
    ConfigError::InvalidPort(port) => {
        println!("Invalid port number: {}", port);
    }
}
```

This is far more robust than matching on strings. The compiler ensures that all
variants are handled correctly, and refactoring the error type will
automatically surface any places that need updating.

## Domain Errors

Errors like `ConfigError` are often called **domain errors**.

A domain error describes something that went wrong within the **problem
domain** of the program. In this case, the domain is configuration loading.

Examples of domain errors include:

* invalid user input
* malformed data
* missing resources
* violated constraints

By modeling these failures explicitly, we make the program easier to reason
about and easier to maintain.

## Modeling Failures Explicitly

This approach reflects a broader Rust philosophy: **make invalid states
unrepresentable** and **make failures explicit**.

Instead of hiding errors behind strings or exceptions, Rust encourages
developers to model them as part of the type system.

The downside is that implementing fully featured error types can involve some
boilerplate, especially when implementing traits like `Display` and
`std::error::Error`.


# 3. Manual Error Implementations

Defining an error enum is only the first step. In Rust, error types are
expected to implement a few standard traits so they integrate well with the
rest of the ecosystem.

At minimum, most error types implement:

* `Debug`
* `Display`

And often they also implement:

* `std::error::Error`

## The `Debug` Trait

The `Debug` trait is primarily used for **developer-facing output**, such as
logging or debugging messages. We already derived it earlier:

```rust
#[derive(Debug)]
pub enum ConfigError {
    MissingFile,
    InvalidPort(u16),
}
```

This allows the error to be printed with `{:?}`.

## The `Display` Trait

The `Display` trait is used for **user-facing error messages**. This is what
gets printed when an error is displayed in a CLI or returned to a user.

Unlike `Debug`, `Display` usually needs to be implemented manually.

Here is what that implementation looks like for our error type:

```rust
use std::fmt;

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::MissingFile => write!(f, "configuration file missing"),
            ConfigError::InvalidPort(p) => write!(f, "invalid port: {}", p),
        }
    }
}
```

The implementation pattern is straightforward:

1. Match on the error variant
2. Write an appropriate error message to the formatter

This gives us clean user-facing messages such as:

```bash
configuration file missing
```

or

```bash
invalid port: 808080
```


## The `std::error::Error` Trait

Many Rust libraries also expect errors to implement the `std::error::Error`
trait. This trait allows errors to participate in **error chains**, where one
error wraps another.

For simple error types, the implementation is often trivial:

```rust
impl std::error::Error for ConfigError {}
```

But when errors wrap other errors, additional methods like `source()` may be required.

## The Boilerplate Problem

None of this code is particularly difficult, but it is repetitive.

For every error type you define, you typically need to write:

* the `enum` definition
* a `Display` implementation
* sometimes an `Error` implementation
* sometimes conversions from other error types

Most of this code follows the same pattern:

* match on the enum
* format a message
* forward errors when necessary

As projects grow, this boilerplate can become tedious and distracting from the
actual logic of the program.

Fortunately, this is exactly the kind of repetitive code that Rust’s **derive
macros** can eliminate.

The `thiserror` crate exists specifically to remove this boilerplate while
keeping error types **fully idiomatic and zero-cost**.


# 4. Enter `thiserror`

As we saw in the previous section, implementing Rust error types manually
involves a fair amount of repetitive code. For every error `enum`, we typically
write:

* the `enum` definition
* a `Display` implementation
* sometimes an implementation of `std::error::Error`
* possibly conversions from other error types

This pattern is so common that the Rust ecosystem has a widely used crate to
eliminate the boilerplate: **`thiserror`**.

`thiserror` is a small crate that provides a **derive macro for error types**.
Instead of writing manual trait implementations, you annotate your error enum
and let the macro generate the code.

Some key characteristics of `thiserror`:

* **Lightweight**: it only provides derive macros and adds no runtime dependencies.
* **Zero runtime cost**: it generates the same code you would write manually.
* **Ideal for library error types**: it helps you create clean, structured, idiomatic error definitions.

Because it only generates standard Rust implementations, using `thiserror` does
not change the behavior of your program. It simply removes the boilerplate.

## A First Example

Let’s rewrite our `ConfigError` type using `thiserror`.

First, import the derive macro:

```rust
use thiserror::Error;
```

Then derive the `Error` trait on the enum:

```rust
#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("configuration file missing")]
    MissingFile,

    #[error("invalid port: {0}")]
    InvalidPort(u16),
}
```

With just these annotations, `thiserror` automatically generates:

* a `Display` implementation
* an implementation of `std::error::Error`

This replaces the manual implementations we wrote earlier.

## The `#[derive(Error)]` Attribute

The `#[derive(Error)]` macro is the core feature of the crate. It tells
`thiserror` to generate the standard trait implementations needed for a Rust
error type.

Under the hood, the macro produces code equivalent to what we would write
manually:

* a `Display` implementation that formats each variant
* an implementation of `std::error::Error`

The result is a fully compliant error type that works with the rest of the Rust
ecosystem.

## The  `#[error("...")]` Attribute

Each enum variant is annotated with an `#[error("...")]` attribute. This
attribute defines the **user-facing error message** associated with that
variant.

For example:

```rust
#[error("configuration file missing")]
MissingFile
```

When this error is displayed, it will produce the message:

```
configuration file missing
```

The attribute also supports formatting arguments. For tuple-style variants,
fields can be referenced by position:

```rust
#[error("invalid port: {0}")]
InvalidPort(u16)
```

Here `{0}` refers to the first field of the variant, which is the invalid port
number.

If the error `InvalidPort(90000)` is displayed, the message becomes:

```
invalid port: 90000
```

With just a few annotations, we now have a clean, fully featured error type
without writing any manual trait implementations.

# 5. Formatting Error Messages

One of the most convenient features of `thiserror` is how it lets you define
error messages directly on each enum variant using the `#[error(...)]`
attribute.

The syntax used inside `#[error("...")]` follows the same formatting rules as
Rust's standard formatting macros such as `format!`, `println!`, and `write!`.
This means you can insert values from the error variant directly into the
message.

This makes error definitions concise, expressive, and easy to read.

## Positional Fields

For tuple-style variants, fields are accessed using **positional indices**.

For example:

```rust
#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("invalid port: {0}")]
    InvalidPort(u16),
}
```

Here the variant `InvalidPort` contains a single field of type `u16`. The
placeholder `{0}` refers to the **first field** of the variant.

If the error value is:

```rust
ConfigError::InvalidPort(90000)
```

then the displayed message will be:

```bash
invalid port: 90000
```

If a variant had multiple fields, you could reference them with `{0}`, `{1}`,
`{2}`, and so on.

For example:

```rust
#[error("invalid range: {0} to {1}")]
InvalidRange(u32, u32)
```

This would allow both values to appear in the formatted message.

## Named Fields

Another option is to use **struct-style variants** with named fields. These can
make error definitions more readable, especially when there are multiple pieces
of data involved.

For example:

```rust
#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("invalid port: {port}")]
    InvalidPort { port: u16 },
}
```

Here the error variant has a named field `port`. Inside the error message, the
field can be referenced directly using `{port}`.

If the error value is:

```rust
ConfigError::InvalidPort { port: 90000 }
```

the message will be:

```bash
invalid port: 90000
```

## Positional vs Named Fields

Both approaches are valid, and the choice is mostly a matter of style and
clarity.

**Tuple-style variants**

```
InvalidPort(u16)
```

are concise and work well when there are only one or two fields.

**Struct-style variants**

```
InvalidPort { port: u16 }
```

can be easier to read when:

* the variant has multiple fields
* the meaning of the fields is not obvious
* the error carries several pieces of context

In larger error types, named fields often make the code easier to maintain.

The ability to embed fields directly into formatted error messages makes
`thiserror` both expressive and ergonomic. But one of its most powerful
features is still ahead: automatic conversion from other error types using the
`#[from]` attribute.


# 6. Automatic Conversions with `#[from]`

So far, our error types have represented **failures specific to our own
domain**. In real programs, however, many errors originate from **other
libraries or the standard library**.

For example:

* file operations may produce `std::io::Error`
* parsing numbers may produce `ParseIntError`
* network operations may produce HTTP errors

A common pattern in Rust is to **wrap these lower-level errors inside your own
error type**. This lets your application expose a single unified error type
while still preserving the original error internally.

This is where one of `thiserror`’s most powerful features comes in: the
`#[from]` attribute.

## Wrapping External Errors

Consider the following error type:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("io error")]
    Io(#[from] std::io::Error),

    #[error("parse error")]
    ParseInt(#[from] std::num::ParseIntError),
}
```

Here, the `AppError` enum includes variants that wrap errors from other parts
of the system.

* `Io` wraps `std::io::Error`
* `ParseInt` wraps `std::num::ParseIntError`

The important part is the `#[from]` attribute attached to the field.

## What `#[from]` Generates

When you annotate a field with `#[from]`, `thiserror` automatically generates
an implementation of the `From` trait.

For example, this line:

```rust
Io(#[from] std::io::Error)
```

generates something equivalent to:

```rust
impl From<std::io::Error> for AppError {
    fn from(err: std::io::Error) -> Self {
        AppError::Io(err)
    }
}
```

This automatic conversion is extremely useful because it allows errors to be
**converted implicitly** when using the `?` operator.

## Error Propagation with `?`

One of the main uses of `Result` in Rust is to propagate errors using the `?`
operator.

For example:

```rust
fn parse_number(input: &str) -> Result<i32, AppError> {
    let x: i32 = input.parse()?;
    Ok(x)
}
```

The `parse()` method returns:

```rust
Result<i32, ParseIntError>
```

But our function returns:

```rust
Result<i32, AppError>
```

Normally this mismatch would cause a compilation error. However, because we
added `#[from] std::num::ParseIntError`, Rust knows how to convert the
`ParseIntError` into `AppError`.

So when `?` encounters an error, it effectively performs:

```bash
ParseIntError -> AppError
```

before returning.

This mechanism allows errors to **flow naturally through layers of a program**
without requiring explicit conversion code.

## A Unified Error Type

Using `#[from]`, it becomes easy to define a **single application error type**
that wraps multiple underlying errors.

For example, a CLI application might need to deal with:

* file system errors
* parsing errors
* configuration errors
* network errors

Instead of returning many different error types, the application can unify them
under a single enum like `AppError`.

Each lower-level error can then be converted automatically using `#[from]`.

## Why This Matters

This pattern provides several important benefits:

* **Cleaner code**: no manual error conversion
* **Better composition**: errors propagate naturally with `?`
* **Clear boundaries**: external errors are wrapped in your own type
* **Unified error handling**: the application deals with one error type

Because of this, `#[from]` is one of the most commonly used features of
`thiserror`.


# 7. Error Sources and Chains

In real applications, errors rarely happen in isolation. Often, one error
occurs **because another error happened first**.

For example:

* a configuration loader fails
* because reading the file failed
* because the file does not exist

This creates a **chain of errors**, where a high-level error wraps a
lower-level one. Rust’s error system supports this idea through the concept of
**error sources**.

The trait `std::error::Error` includes a method called `source()` that allows
an error to expose its underlying cause.

```rust
fn source(&self) -> Option<&(dyn std::error::Error + 'static)>
```

If an error wraps another error, `source()` returns it. Otherwise it returns
`None`.

This mechanism allows tools and libraries to **walk the chain of failures**,
which is extremely useful for debugging.

## Defining an Error Source

`thiserror` makes it easy to define error sources using the `#[source]`
attribute.

For example:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("failed to read config")]
    Read {
        #[source]
        source: std::io::Error,
    },
}
```

Here the `Read` variant represents a high-level failure: the configuration
could not be read.

However, the real cause is stored in the `source` field, which contains a
`std::io::Error`.

The `#[source]` attribute tells `thiserror` that this field represents the
**underlying cause** of the error.

## What `#[source]` Does

When `#[source]` is applied to a field, `thiserror` automatically implements
the `source()` method from the `std::error::Error` trait.

Conceptually, it generates something like:

```rust
fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
    match self {
        ConfigError::Read { source } => Some(source),
    }
}
```

This allows the underlying error to be retrieved and inspected by
error-handling tools.

## Error Chains in Practice

Suppose our configuration loader fails because the file does not exist.

The error chain might look like this:

```bash
failed to read config
└── No such file or directory (os error 2)
```

The top-level error describes the **high-level context** ("failed to read
config"), while the source error contains the **low-level system failure**.

This layered structure is extremely helpful when diagnosing problems,
especially in larger applications where errors may pass through many layers of
code.

## Why Error Chains Matter

Error chains provide several important benefits:

* **Better debugging**: Developers can see both the high-level context and the underlying cause of a failure.
* **Clear separation of concerns**: Each layer of the program can add context without losing the original error.
* **Better tooling**: Logging frameworks and error-reporting tools can traverse the entire error chain automatically.

## `#[from]` vs `#[source]`

It’s worth noting that `#[from]` (introduced in the previous section)
**implicitly marks a field as a source**.

For example:

```rust
Io(#[from] std::io::Error)
```

is equivalent to:

```rust
Io {
    #[source]
    source: std::io::Error
}
```

with the added bonus that `#[from]` also generates the `From` implementation.

In other words:

* `#[source]` indicates **an underlying error**
* `#[from]` indicates **an underlying error + automatic conversion**

Both attributes help create structured error chains that make Rust applications
easier to debug and maintain.


# 8. Transparent Errors

Sometimes an error type exists mainly to **wrap another error without changing
its message**. In these cases, writing a new error message would add little
value and might even hide useful information from the underlying error.

`thiserror` provides a convenient way to express this pattern using the
`#[error(transparent)]` attribute.

For example:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error(transparent)]
    Io(#[from] std::io::Error),
}
```

Here, the `Io` variant wraps a `std::io::Error`, but the
`#[error(transparent)]` attribute tells `thiserror` to **forward the display
message of the wrapped error directly**.

In other words, the error message comes from the underlying `std::io::Error`,
not from the wrapper.

If the wrapped error prints something like:

```bash
No such file or directory (os error 2)
```

then the `AppError::Io` variant will display exactly the same message.

## What “Transparent” Means

A transparent error essentially says:

> “This error is just a wrapper. Use the inner error's message.”

Without `transparent`, you might define something like:

```rust
#[error("io error")]
Io(#[from] std::io::Error)
```

This would produce messages such as:

```bash
io error
```

While this might be acceptable in some contexts, it also **loses the more
detailed message** that the underlying error might contain.

Using `transparent` preserves the full message from the original error while
still allowing your application to wrap it inside its own error type.

## When to Use Transparent Errors

Transparent errors are useful in a few common situations.

**Wrapper Error Types**

Sometimes you want to expose a **single application error type** that
aggregates errors from different parts of the system.

However, some of those errors already have good messages, and you don't want to
rewrite them.

For example:

```rust
#[derive(Debug, Error)]
pub enum AppError {
    #[error("configuration error")]
    Config(ConfigError),

    #[error(transparent)]
    Io(#[from] std::io::Error),
}
```

Here:

* configuration errors get their own message
* I/O errors pass through unchanged

**Pass-Through Errors**

Transparent errors are also useful when a layer of your program simply
**forwards errors from a lower layer**.

For example, a higher-level module might expose a unified error type but still
want the original error message to reach the user.

In these cases, transparent errors let you **preserve the original message
while still structuring your error types properly**.

## A Good Rule of Thumb

A useful guideline is:

* Use a **custom error message** when you want to add context.
* Use **`#[error(transparent)]`** when the underlying error message already provides the right information.

This balance allows your error types to remain structured and expressive while
avoiding unnecessary duplication.


# 9. Designing Real Error Types

So far we’ve looked at the mechanics of `thiserror`: formatting messages,
wrapping other errors, and building error chains. The next step is learning how
to **design good error types for real applications**.

A well-designed error type should describe the **meaningful failures of your
program**, while still allowing lower-level errors to propagate when necessary.

In Rust, the most common pattern is to use an **enum that groups all the
relevant failures for a particular component or application**.

For example, a command-line application might define an error type like this:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum CliError {
    #[error("config file missing")]
    ConfigMissing,

    #[error("invalid port: {0}")]
    InvalidPort(u16),

    #[error("io error")]
    Io(#[from] std::io::Error),
}
```

This enum represents the different ways the CLI can fail.

## Domain Errors vs Infrastructure Errors

A useful mental model when designing error types is to distinguish between
**domain errors** and **infrastructure errors**.

**Domain errors** represent failures that are specific to the logic of your
application.

Examples include:

* missing configuration
* invalid user input
* invalid command arguments
* malformed data

In the example above:

```rust
ConfigMissing
InvalidPort(u16)
```

are domain errors. They describe problems related to the **meaning of the
program’s inputs**.

**Infrastructure errors**, on the other hand, come from the systems your
program interacts with.

Examples include:

* file system errors
* network errors
* database errors
* serialization errors

These are typically errors produced by other libraries or the standard library.

In our example:

```rust
Io(#[from] std::io::Error)
```

wraps an infrastructure error originating from the operating system.

## Grouping Failures in a Single Error Type

A good application error type usually **groups both kinds of failures**:

* domain-specific problems that your program defines
* lower-level errors that come from external libraries

This gives you a single error type that represents everything that can go wrong
in a particular layer of your program.

For example, a CLI application might fail because:

* the configuration file is missing
* the configuration contains an invalid value
* the file system operation fails

All of these failures can be represented in one `enum`.

This has several advantages:

* **Clear API boundaries**: Functions can return `Result<T, CliError>` instead of many different error types.
* **Better structure**: Callers can match on domain errors when they want special handling.
* **Natural error propagation**: Infrastructure errors can propagate automatically using `#[from]`.

## Adding Context at Higher Levels

As errors move through layers of a program, higher-level code can add **additional context**.

For example:

* a configuration module may define `ConfigError`
* a CLI module may wrap it inside `CliError`

Each layer can describe the failure in terms of its own responsibilities while
preserving the underlying cause.

This layered design leads to **clearer error messages and better debugging
information**.

## A Simple Guideline

When designing error types, a useful guideline is:

* define **domain errors explicitly**
* wrap **external errors using `#[from]`**
* group related failures into a **single enum per subsystem**

Following this pattern keeps error handling structured and maintainable,
especially as a codebase grows.


# 10. `thiserror` vs `anyhow`

REDO AFTER 6:29PM

10. thiserror vs anyhow

Important conceptual section.

Explain difference:

crate	use case
thiserror	library error types
anyhow	application error handling

Example:

Library:

pub fn parse_port(s: &str) -> Result<u16, ConfigError>

Application:

fn main() -> anyhow::Result<()> 

# 11. Best Practices

REDO AFTER 6:29PM

11. When NOT to Use thiserror

Teach judgment.

Cases:

quick scripts

throwaway code

binaries using anyhow

# 12. Final Example

REDO AFTER 6:29PM

12. Final Example: Real CLI Error Type

Combine everything.

Example:

#[derive(Debug, Error)]
pub enum CliError {
    #[error("configuration file missing")]
    ConfigMissing,

    #[error("invalid port: {0}")]
    InvalidPort(u16),

    #[error("io error")]
    Io(#[from] std::io::Error),

    #[error("parse error")]
    ParseInt(#[from] std::num::ParseIntError),
}

Then show a real function using ?.
