---
layout: "post"
title:  "How Procedural macros actually work"
date:   "2025-12-25 11:23:35 +0530"
categories: rust
--- 

Ever wondered how Rust’s procedural macros actually work? In this blog, we’ll get into the weeds a bit. I’ve been working on improving proc-macro expansion efficiency in rust-analyzer, and thought it might be interesting to share some of the things I learned along the way.

## Macros

Macros are everywhere in Rust, even though plenty of languages get by just fine without them. So what exactly are macros, and why do they matter?

At a high level, macros exist to do three main things:

* Let you write code that generates other code.
* Extend the language with custom syntax and patterns.
* Cut down on repetitive boilerplate.

## How macros generate new code

One of the superpowers of macros is that they let you write code that *produces more code*. A simple way to see this is with logging.

Imagine you want to print a value along with some extra context:

```rust
fn main() {
    let x = 42;
    std::io::stdout()
        .write_fmt(format_args!("x = {}\n", x))
        .unwrap();
}
```

That works, but it’s not exactly pleasant to write every time. Instead, we usually reach for:

```rust
fn main() {
    let x = 42;
    println!("x = {}", x);
}
```

That `println!` isn’t a function call, it’s a macro invocation. At compile time, the macro expands into code similar to the first example.

Under the hood, `println!` is a declarative macro. A highly simplified version looks something like this:

```rust
macro_rules! println {
    ($($arg:tt)*) => ({
        std::io::stdout()
            .write_fmt(format_args!($($arg)*))
            .unwrap();
    })
}
```

Here, `$($arg:tt)*` is the macro pattern. It matches *any number of tree tokens*, which allows `println!` to accept flexible input like format strings and arguments.

When you write:

```rust
println!("x = {}", x);
```

the compiler matches `"x = {}", x` against the macro pattern and expands it into the corresponding Rust code before type checking even begins.

## How macros create new syntax

Macros don’t just generate code, they can also *feel* like they introduce entirely new syntax.

A good example of this is the `sql!` procedural macro used in some database libraries. It lets you write SQL directly inside Rust code and have it checked and transformed at compile time.

Here’s what a macro invocation might look like:

```rust
sql! {
    SELECT id, name
    FROM users
    WHERE active = true
}
```

That doesn’t look like Rust at all. Yet it’s perfectly valid Rust code, because `sql!` is a procedural macro.

What’s happening is that the macro receives the tokens inside its braces, interprets them using SQL-like rules, and then generates normal Rust code. After expansion, the compiler only sees Rust, something along the lines of:

```rust
Query::new("SELECT id, name FROM users WHERE active = true")
    .bind(true)
```

From the compiler’s perspective, there’s no SQL involved anymore, just plain Rust code that it knows how to type-check and compile.

This is how macros can effectively embed small, domain-specific languages inside Rust.

One important limitation to keep in mind is that macros don’t get access to raw source text. Whitespace is mostly irrelevant, and everything is passed in as tokens. That’s why you can’t embed indentation-sensitive languages like Python inside a macro.

You’ll see similar ideas in other places too:

* Collection literals like `vec![1, 2, 3]`.
* Formatting macros such as `println!` and `format!`.
  (`println!` itself expands to another compiler-built macro, `format_args_nl!`.)

In short, procedural macros can *reshape* Rust syntax, but only using tokens that are already valid in the language.

## Procedural macros

Essentially, a procedural macro is a Rust function executed at compile time. Such functions belong to a special crate marked with the `proc-macro` flag. In `Cargo.toml`, this looks like the following:

```toml
[package]
name = "my-proc-macro"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true
```

## Types of procedural macros

There are three types of procedural macros:

* Function-like procedural macros
  These macros are declared using the `#[proc_macro]` attribute and called like regular functions, similar to declarative macros:
  
  ```rust
  #[proc_macro]
  pub fn foo(body: TokenStream) -> TokenStream { ... }
  ...
  foo!(foo bar baz)
  ```
* Custom derive procedural macros
  These macros are declared using the `#[proc_macro_derive]` attribute and are used in `#[derive]` for structures and enums:
  ```rust
  #[proc_macro_derive(Bar)]
  pub fn bar(body: TokenStream) -> TokenStream { ... }
  ...
  #[derive(Bar)]
  struct S;
  ```
* Custom attributes
  These macros are declared using #[proc_macro_attribute] and are called as items attributes:
  ```rust
  #[proc_macro_attribute]
  pub fn baz(
    attr: TokenStream,
    item: TokenStream
  ) -> TokenStream { ... }
  ...
  #[baz]
  fn some_item() {}
  ```

## Procedural macros API

### Procedural macro body

Let's first clarify what a procedural macro body is. In the case of a function-like macro, the body is everything between the first brackets:

```rust
#[proc_macro]
pub fn foo(body: TokenStream) -> TokenStream { ... } ------>  foo! (foo bar baz)
           -----------------                                        -----------
```

In the case of a custom derive macro, the body is the whole attributed structure:

```rust
#[proc_macro_derive(Bar)]                                                   #[derive(Bar)]
pub fn bar(body: TokenStream) -> TokenStream { ... } ------------------>     struct S;
           -----------------                                                 ---------
```
For an attribute macro, the body includes the whole item ( `fn some_item() {}` ). There can also be more parts for the macro body in the attribute itself (they are passed as additional attributes to the function as well):

```rust
#[proc_macro_attribute]
pub fn baz (                                                               #[baz(qux, quz)]
    attr: TokenStream,                                                     fn some_item() {}
    item: TokenStream
) -> TokenStream { ... }
```
To illustrate this, we'll examine an identity macro, which simply returns the body that it takes, without doing anything else:

```rust
extern crate proc_macro;
use proc_macro::TokenStream;

#[proc_macro]
pub fn foo(body: TokenStream) -> TokenStream {
    return body
}
```

Suppose we have a program that calls `hello()`, where hello is inside a `foo!` macro. In this situation, the `foo` macro will be expanded in such a way that it will look like there was no macro in the first place:

```rust
use my_proc_macro::*;

// foo! {
    fn hello() {
        println!("Hello, world!");
    }
//}

fn main() {
    hello();
}
```

Similarly, this could be written with an attribute macro:

```rust
extern crate proc_macro;
use proc_macro::TokenStream;

#[proc_macro_attribute]
pub fn baz(attr: TokenStream, item: TokenStream) -> TokenStream {
    return item
}
...
use my_proc_macro::*;

#[baz]
fn hello() {
    println!("Hello, world!");
}

fn main() {
    hello();
}
```

## Tokens, TokenStream and TokenTree

The body of a procedural macro is divided into pieces called tokens. A token is a string of a particular type, which is assigned to it during the parsing of a macro body. There are three types of tokens: identifiers, punctuation symbols, and literals.

Procedural macros operate with special data types from the `proc_macro` crate, which is a part of the standard library and is linked automatically when procedural macro are compiled. One of these special types, `TokenTree`, represents the enum of the possible token types:

```rust
struct TokenStream(Vec<TokenTree>);
enum TokenTree {
    Ident(Ident),
    Punct(Punct),
    Literal(Literal)
}
```
Another data structure, `TokenStream`, represents the list of tokens and allows you to iterate the token list (`body.into_iter()`):

```rust
#[proc_macro]
pub fn foo(body: TokenStream) -> TokenStream {
    for tt in body.into_iter() {
        match tt {
            TokenTree::Ident(_) => eprintln!("Ident"),
            TokenTree::Punct(_) => eprintln!("Punct"),
            TokenTree::Literal(_) => eprintln!("Literal"),
            _ => {}
        }
    }
    return TokenStream::new();
}
```
There one more enum variant in the `TokenTree`, which is called `Group`:

```rust
enum TokenTree {
    Ident(Ident),
    Punct(Punct),
    Literal(Literal),
    Group(Group)
}
```

Groups appear when the parser encounters brackets. The brackets that form a group can be either round, square, or braces.

For example, a macro with the following body

```rust
foo! ( foo { 2 + 2 } bar );
```
will be parsed into two identifiers(`foo` and `bar`) and a group(`{2 + 2}`). A group here includes braces and another `TokenStream` (literals `2` and `2` and a punctuation symbol `+`). We can see that `TokenStream` is not strictly a stream. It's a kind of tree, where each node is formed by brackets and the leaves represent singular tokens.

## Spans

Why are TokenStream and TokenTree necessary for the procedural macros API? Why aren't raw strings enough? We can think of this kind of code(which will not work):

```rust
#[proc_macro]
pub fn foo(body: String) -> String {
    format!("foo({})", body)
}
```

To understand why the code doesn't work, we need to go back to the token structure.

Beside the type and the actual string of symbols, a token structure also includes a `Span`:

```rust
enum TokenTree {
    Ident(Ident) ----------------------------- struct Ident(String, Span);
                                                                    ----
    Literal(Literal) -------------------------- struct Literal(String, Span);
                                                                       ----
    Punct(Punct) ----------------------------- struct Punct (
                                                      char,
                                                      Spacing,
                                                      Span
                                                );
    Group(Group)
}
```

Span contains information about where in the original code the token was placed.This is necessary for the compiler to highlight the errors correctly.

For example, we can take the same macro and intentionally pass a string instead of a numeric value into a call:

```rust
fn main() {
    foo!(1, "");
}

fn foo(a: i32, b: i32) {}
```

Since the function expects an `i32` value, the compiler will report an error. But where will the compiler report the error? It will be shown at the token where the error would be expected if it were a regular function call, not at the whole macro call.

This is possible because we passed the whole `TokenStream` into the expansion, and each token contains a `Span`. Span informs the compiler that this particular code fragment should be mapped to that particular fragment in macro expansion. This way, the compiler can map the errors that occur during the compilation of the expanded code.

Now to summarize, a procedural macro structure is built from the following blocks:
* TokenStream, which is a vector of TokenTrees.
* TokenTree is an enum of 3 token types plus a Group
* A Group is formed by brackets.
* Each token has a Span, which is used for error mapping.