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

## Compilation of procedural macros

To start, let's see how we might write `Hello, world` program using a separate crate inside a procedural macro:

> proc-macro-example/Cargo.toml
```toml
[package]
name = "proc-macro-example"
edition = "2024"

[lib]
path = "proc-macro-example"
```

> proc-macro-example/main.rs
```rust
extern crate proc_macro;
use proc_macro::TokenStream;

#[proc_macro]
pub fn foo(body:  TokenStream) {
    body
}
```

> hello/Cargo.toml
```toml
[package]
name = "hello"
version = "0.1.0"
edition = "2024"

[dependencies.proc-macro-example]
path = "proc-macro-example"
```
> hello/main.rs
```rust
use proc_macro_example::foo;

foo! {}

fn main() {
    println!("Hello, World!");
}
```

In order to build this project, Cargo performs 2 calls to rustc:

```ini
cargo build -vv

rustc
    --crate-name proc-macro-example
    --crate-type proc-macro
    --out-dir ./target/debug/deps
    proc-macro-example/src/lib.rs

rustc
    --crate-name hello
    --crate-type bin
    --out-dir ./target/debug/deps
    --extern proc-macro-example = ./target/debug/deps/libproc_macro_example-547aff81ea3cda77.so
    src/main.rs
```
During the first call to rustc, Cargo passes the procedural macro source with the `--crate-type proc-macro` flag. As a result, the compiler creates a dynamic library, which will be then passed to the second call along with the initial `Hello, World` source to build the actual binary. We can see that in the last line of the second call:
```ini
        --extern proc-macro-example = ./target/debug/deps/libproc_macro_example-547aff81ea3cda77.so
    src/main.rs
```

This is how the process can be illustrated.

Here's the first call to rustc:

![alt text](<../assets/posts/Screenshot from 2025-12-27 12-28-51.png>)

Here's the second call to rustc:
![alt text](<../assets/posts/Screenshot from 2025-12-27 12-34-36.png>)
![alt text](<../assets/posts/Screenshot from 2025-12-27 12-35-46.png>)
![alt text](<../assets/posts/Screenshot from 2025-12-27 12-36-20.png>)
![alt text](<../assets/posts/Screenshot from 2025-12-27 12-37-14.png>)

## First call to rustc: dynamic library

The intermediary dynamic library includes a special symbol, `__rustc_proc_macro_decls_***_`, which contains an array of macros declared in the project.

This is what a procedural macro's code could look like after certain changes that rustc makes during compiling:
```rust
extern crate proc_macro;
use proc_macro::TokenStream;

pub fn foo(body: TokenStream) -> TokenStream {
    body
}

use proc_macro::bridge::client::ProcMacro;
#[no_mangle]
static __rustc_proc_macro_decls_2db78d2d43c54ae0__: &[ProcMacro] = &[
    ProcMacro::bang("foo", crate::foo)
];
```

The `ProcMacro` array includes the information on macro types and names, as well as references to procedural macro functions (`crate::foo`).

The `__rustc_proc_macro_decls_***__` symbol is exported from the dynamic library, and rustc finds it during the second call.

## Second call to rustc: ABI 

During the second call, rustc finds the `__rustc_proc_macro_decls_***_` symbol and recognizes a function reference there.

At this point, we might expect the compiler to command the dynamic library to expand the macro using the given TokenStream:

![alt text](<../assets/posts/Screenshot from 2025-12-27 13-16-56.png>)

However, this can't be done due to the fact that Rust doesn't have a stable ABI. An ABI - Application Binary interface - includes calling conventions (like the order in which structure fields are placed in memory). In order for a function to be called from a dynamic library, its ABI should be known to the compiler.

Rust's ABI is not fixed, meaning that it can change with each new compiler version. So there are two requirements for dynamic library's ABI and the program's ABI to match:

1. Both the library and the program should be compiled using the same compiler version.

2. The codegen backend of the compiler should also be the same.

As we know, rustc is compiled by itself. When a new compiler version is release, it is first compiled by the previous version, and then compiled by itself. Similarly, in the case of our procedural macro library, it is compiled by the compiler, which it is then linked into. SO it seems that rustc and procedural macros are compiled using the same compiler version, and their ABI's should match. But here comes the second of the equation - the codegen backend.


**Rustc codegen backend and proc macros**

This is where it's helpful to take a look at how things used to work. Prior to 2018, Rust's ABI had been used for procedural macros, but then it was decided that the compiler's backend should be modifiable. Today, although the default rustc backend is LLVM-based, there are alternative builds like Cranelift or others - with a GCC backend.

How is it possible to add more backends to the compiler while simultaneously keeping procedural macros working?

There are two solutions that seem to be obvious, but they have their flaws:

* C ABI

In the case of C ABI, a function should be prepended with `extern "C"`:

`extern "C" fn foo() { ... }`

This kind of function can take C types or the types declared with the `repr(C)` attribute:

`#[repr(C)] struct Foo { ... }`

* Full serialization of macro-related types

In this case, macro-related types like TokenStream would be serialized and then passed to the dynamic library. The macro would de-serialize them back to their inner types, call the function, and then serialize the result to pass them back to the compiler:

![alt text](<../assets/posts/Screenshot from 2025-12-27 13-47-18.png>)

But it is not that simple. The correct solution came in pull request [#49219](https://github.com/rust-lang/rust/pull/49219) called "Decouple proc_macro from the rest of the compiler". The actual process adopted in rustc is as follows:

1. Before rustc calls a procedural macro, it puts the TokenStream into the table of handles with a specific id. Then the procedural macro is called with that id instead of a whole data structure.

2. On the dynamic library's side, the id is wrapped into the library's inner tokenStream type. Then, when the method of that TokenStream is called, that call is passed back to rustc with the id and the method name.

3. Using the id, rustc takes the original tokenStream from the table of handles and calls the method on it. The result goes to the table of handles and gets the id that is used for further operations on that result, and so on.

![alt text](<../assets/posts/Screenshot from 2025-12-27 13-58-15.png>)

How is this approach better than the simpler version with full serialization of all structures or C ABI?

* Spans link a lot of compiler inner types, which are better left unexposed. Also, implementation of serialization for all of those inner types would be expensive.
* This approach allows backcalls (the procedural macro might want to ask the compiler for some additional action).
* With this approach, macros can (in theory) be extracted into a separate process or be executed on a virtual machine.

## Proc macros and potentially dangerous code

As we saw, a procedural macro is essentially a dynamic library linked to the compiler. That library can execute arbitrary - and potentially dangerous - code. For example, that code could segfault, causing the compiler to segfault too, or call system fork and duplicate the compiler.

Aside from this apparently dangerous behavior, procedural macros can also make other system calls, such as accessing a file system or the web. These calls are not necessarily unsafe, but might not be good practice in the first place.

## Procedural macros in rust-analyzer

In order for the rust-analyzer to analyze procedural macros on the fly, it needs to have the code of expansion on hand all the time.

In case of declarative macros, the LSP can expand macros by itself. But in the case of procedural macros, it needs to load the dynamic library and perform the actual macro calls.

One approach might be to simply replace the compiler with the LSP, keeping the same workflow as described above. However, bear in mind that a procedural macro can segfault (and cause the LSP to segfault) or, or example, occupy a lot of memory, and the LSP will fail with out-of-memory errors. For these reasons, procedural macro expansion must be extracted into another process separate from the LSP.

That process is called the `expander`. It links the dynamic library and uses the same interface as the compiler to interact with procedural macros. Communication between the LSP and expander is performed using full data serialization.


![alt text](<../assets/posts/Screenshot from 2025-12-27 14-17-17.png>)

---

And this is how procedural macros work!!!