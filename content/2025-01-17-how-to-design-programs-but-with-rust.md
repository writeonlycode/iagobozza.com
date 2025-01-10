+++
title = "How to Design Programs, but with Rust"
description = "In this series of posts, I try to use the strategies presented in the book How to Design Programs, by Matthias Felleisen, Robert Bruce Findler, Matthew Flatt and Shriram Krishnamurthi with the Rust programming language."
date = 2025-01-17
authors = ["Iago Bozza"]
draft = true

[taxonomies]
tags = ["rust", "how_to_design_programs_with_rust"]

[extra]
og_image = "/how-to-desing-programs-with-rust/htdp-2e-cover.gif"
+++

The book [How to Design Programs: An Introduction to Programming and
Computing](https://mitpress.mit.edu/9780262534802/how-to-design-programs/), by
Matthias Felleisen, Robert Bruce Findler, Matthew Flatt and Shriram
Krishnamurthi, presents a systematic way to write programs. It focus on design
recipes, templates and iterative refinement: the design recipes and templates
help to deal with the simple cases, and through iterative refinement we can
refine the simple cases until we can handle the complex ones. In this series of
posts, I try to use the strategies presented there to write programs, but using
the Rust programming language instead of their *SL languages.

<!-- more -->

<!--{{ image(src="/how-to-desing-programs-with-rust/htdp-2e-cover.gif", alt="How to Desing Programs Cover", position="center", style="border-radius: 8px;") }}-->


# 1. The General Recipe

The general recipe to design functions and programs, which we can find in the
prologue, is:

> 1. **From Problem Analysis to Data Definitions**: Identify the information
>    that must be represented and how it is represented in the chosen
>    programming language. Formulate data definitions and illustrate them with
>    examples.
> 2. **Signature, Purpose Statement, Header**: State what kind of data the
>    desired function consumes and produces. Formulate a concise answer to the
>    question what the function computes. Define a stub that lives up to the
>    signature.
> 3. **Functional Examples**: Work through examples that illustrate the
>    function’s purpose. 
> 4. **Function Template**: Translate the data definitions into an outline of
>    the function.
> 5. **Function Definition**: Fill in the gaps in the function template.
>    Exploit the purpose statement and the examples.
> 6. **Testing**: Articulate the examples as tests and ensure that the function
>    passes all. Doing so discovers mistakes. Tests also supplement examples in
>    that they help others read and understand the definition when the need
>    arises - and it will arise for any serious program.

When we apply this general recipe in the context of the Rust programming language, it becomes this:


## 1.1. From Problem Analysis to Data Definitions

Since Rust has a type system, we use it to express how we want to represent
information as data. Let's say we want a program that converts temperatures
from Farenheight to Celcius. Then, we need to represent temperatures:

```rust
type Temperature = f64;
```

The type alias above communicates that we are going to use `f64` to represent
temperatures.

## 1.2. Signature, Purpose Statement, Header

Next, we need to indicate the inputs and outputs of the function, as well as
its purpose. We again leverage the type system to communicate what are the
parameters and the return value of the function. We also include a comment that
describes what the function does:

```rust
// Converts a temperature from Farenheight to Celcius
fn convert_f_to_c(input: Temperature) -> Temperature {
    0.0
}
```

The definition above, together with the comment, communicates that the function
takes a temperature and returns another temperature, as well as that the
purpose of the function is to convert temperatures in Farenheight to
temperatures in Celcius.

## 1.3. Functional Examples

Next, we need to give some examples of inputs and their corresponding outputs.
This time, we leverage Rust's testing framework:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_convert_f_to_c() {
        assert_eq!(convert_f_to_c(-20.0), -28.88888888888889);
        assert_eq!(convert_f_to_c(0.0), -17.77777777777778);
        assert_eq!(convert_f_to_c(10.0), -12.222222222222223);
    }
}
```

The above tests give an idea of the values we should expect for a few selected
inputs. At this point, the program should run, but the tests should fail!

## 1.4. Function Template

Now we start to deal with the body of the function. The first step is to start
with a function template that is appropriate for the data that the function is
processing. We'll look at the templates in the next section, but for now, the
template looks like this:

```rust
// Converts a temperature from Farenheight to Celcius
fn convert_f_to_c(input: Temperature) -> Temperature {
    todo!("{}", input)
}
```

## 1.5. Function Definition

Finally, we fill in the details of the function definition. We want to convert
temperatures in Farenheight to Celcius, so the conversion formula is a good
starting point:

$$F = \dfrac{5}{9}(C -32) $$

In Rust, it'll look like this:


```rust
// Converts a temperature from Farenheight to Celcius
fn convert_f_to_c(input: Temperature) -> Temperature {
    (input - 32.0) * (5.0 / 9.0)
}
```

## 1.6. Testing

Finally, run the tests with `cargo test`! At this point, the program should run
and the tests should pass! If a test still fails, then either you made a
mistake when writing the tests, or you made a mistake when writing the
function, or both!

# 2. World Programs

The previous function was a batch program. Now we develop an interactive
graphical program. We need at least the following: a data definition that
represents the state of the world (`ws`), a function that takes this state and
renders it (`render`). Optionally, we may also have: a function that updates
the state at each click (`clock_tick_handler`), a function that updates the
state in response to keyboard events (`keystroke_handler`), a function that
updates the state in response to mouse events (`mouse_event_handler`), and a
function that checks whether the program should exit (`is_end`).

```rust
type Position = f32;
```
