---
layout: "post"
title:  "Type-level Programming in Rust"
date:   "2050-01-12 14:23:35 +0530"
categories: rust
--- 

Typestate is the concept of encoding state machines in a programming language's type system. While not specific to Rust, typestate has been explored at length in the context of Rust. Typestate boils down to four ideas:

1. Each state is represented as a unique type.
2. State transitions are only available as methods for corresponding state type.
3. Taking a state transition returns a state machine of the new state type.
4. State transitions invalidate old state.

For example, here's a state machine for a send-then-receive channel:

```rust
// Each state is a unique type
struct Receiving;
struct Sending;

// The state machine is parameterized by the state
#[repr(transparent)]
struct Channel<State> {
    channel: ...,
    _state: PhantomData<State>
}

// Methods for the state are uniquely associated with only the state
impl Channel<Receiving> {
    // recv consumes ownership, ensuring old state is invalidated
    fn recv(mut self) -> (Channel<Sending>, String) {
        let msg = self.chan.recv();
        // The state type changes after executing a transition
        (unsafe { transmute(self) }, self)
    }
}

impl Channel<Sending> {
    fn send(mut self, msg: String) -> Channel<Receiving> {
        self.chan.send(msg);
        unsafe { transmute(self) }
    }
}

#[test]
fn channel_test() {
    let c: Channel<Sending> = Channel::new();
    let c: Channel<Receiving> = c.send("hi");
    let (c, msg) = c.recv();
    // and so on
}
```

This pattern works effectively for simple finite state machines, where the logic to determine the next state is straightforward. In this note, we will explore situations where determining the next state is not so simple. In the process, we'll talk about type-level programming, or how you can use Rust's type system to encode **computations on types**.

> Part of the goal of this note is to show the value of type-level programming in practice. These same mechanisms have already been used for more esoteric purposes like showing Rust's type system is turing complete, but I think type-level programming can really help us design better systems

## 1. Information flow control

As a first example, consider a basic information flow control problem. In our program we have low security values(anyone can read them) and high security values (only authorized users can read them).

We represents this idea like so:

```rust
// Each security level is a type
struct HighSec;
struct LowSec;

// An Item wraps an arbitrary type T, associating it with a Level
struct Item<T, Level> {
    t: Box<T>,
    _marker: PhantomData<Level>
}

// Constructors for building items of a particular security
impl<T> Item<T, LowSec> {
    pub fn low_sec(t: T) -> Item<T, LowSec> {
        Item { t: Box::new(t), _marker: PhantomData }
    }

    pub fn high_sec(t: T) -> Item<T, HighSec> {
        Item { t: Box::new(t), _marker: PhantomData }
    }
}

// For simplicity, a naked Item can be read by anyone
impl<T, Level> Deref for Item<T, Level> {
    type Target = T;
    fn deref(&self) -> &T {
        &self.t
    }
}
```

We would like to have a vector of these items with the following property:

* If all of the items are low security, anyone can read any item.
* If any of the items are hight security, only an authorized user can read any item.

For example, our vector should pass this test:

```rust
let v = SecureVec::new();
let lo = Item::low_sec(1);
let hi = Item::high_sec(2);
let v = v.push(lo);         // v is still low sec
assert_eq!(*v.get(0), 1)    // ok to read v

let v = v.push(hi);         // v is now high sec
// assert_eq!(v.get(0), 1); // can't read anymore, compiler error

let w = HighSecWitness::logic();
assert_eq!(*v.get_secure(1, w), 2) // can read after login
```

A basic type state attempt looks like this. We can create and read a low-security vendor:

```rust
#[repr(transparent)]
struct SecureVec<T, Level> {
    items: Vec<Item<T, Level>>,
    _marker: PhantomData<Level>
}

impl<T> SecureVec<T, LowSec> {
    pub fn new() -> SecureVec<T, LowSec> {
        SecureVec {items: Vec::new(), _marker: PhantomData}
    }

    pub fn get(&self, i: usize) -> &T {
        &self.items[i]
    }
}
```

And we can protect a high-security vector through a witness:

```rust
struct HighSecWitness;
impl HighSecWitness {
    // sprinkle some high-security authentication in here...
    pub fn login() -> HighSecWitness {
        HighSecWitness
    }
}

impl<T> SecureVec<T, HighSec> {
    pub fn get_secure(&self, i: usize, _witness: HighSecWitness) -> &T {
        &self.items[i]
    }
}
```

Now, to the main idea: how can we implement `push`? There are four possible state combinations: a high/low security vector with a high/low security item. While we can implement each combination as a separate method, it's simpler to consider the underlying logic. `push` should return a vector of level `max(vec_level, item_level)` where `max(hi, lo) = hi`.

Our goal is to encode `max` as a type-level computation, i.e. an operator on types. The high-level idea:

* Traits definitions are function signature from types to types.
* Trait type parameters represent inputs and associated types represent outputs.
* Trait implementations define individual mapping from inputs to outputs.

Here are those ideas in action to compute the max security level:

```rust
// Self (implicitly) is the left operant, Other is the right operant,
// and Output is the output
trait ComputeMaxLevel<Other> {
    type Output;
}

// These impls define the core computation
impl ComputeMaxLevel<LowSec> for LowSec { type Output = LowSec; }
impl ComputeMaxLevel<HighSec> for LowSec { type Output = HighSec; }
impl ComputeMaxLeveL<LowSec> for HighSec { type Output = HighSec; }
impl ComputeMaxLevel<HighSec> for HighSec { type Output = HighSec; }

// the type alias gives us a more convenient way to "call" the type operator
type MaxLevel<L, R>  = <L as ComputeMaxLevel<R>>::Output
```

> The most confusing part is the `MaxLevel` alias. In brief: `L as ComputeMaxLevel<R>` says "treat `L` as the trait object `ComputeMaxLevel<R>`". This is necessary since mulitple computation traits may have associated `Output` with `L`, so the explicit cast disambiguates the `MaxLevel` computation from the rest.

Here's an example of using the type operator:

```rust
let _: MaxLevel<HighSec, LowSec> = HighSec; // ok
let _: MaxLevel<LowSec, LowSec> = LowSec; // ok
let _: MaxLevel<LowSec, LowSec> = HighSec; // type error
```

Now, we can implement `SecureVec::push` in one method:

```rust
impl<T, VecLevel> SecureVec<T, VecLevel> {
    pub fn push<ItemLevel>(mut self, t: Item<T, ItemLevel>) -> SecureVec<T, MaxLevel<ItemLevel, VecLevel>>
    where 
    ItemLevel: ComputeMaxLevel<VecLevel>{
        unsafe {
            self.items.push(transmute(t));
            transmute(self)
        }
    }
}
```

Notice the usage of `MaxLevel` in the return of `push`. This is the key use of the type operator as a type-level computation. The other main component is the `where` clause: when used generically (over any possible `ItemLevel`), we have to use a trait bound to ensure that `ComputeMaxLevel` can be "called" on `ItemLevel`.

Excellent! We've now use a type-level computation to more abstractly specify typestate in our information flow control API. Next, we'll look at an example with a more complex type-level program.

## 2. Two-party communication protocols

When two parties synchronously communicate with each other (e.g a client and server exchanging information), that communication protocol can be modeled as a session type. We're going to look at session types implemented in rust. We will focus on the aspect of session types that showcase type-level programming.

Session types are a domain-specific language of state machines, described by this grammar:

```

Session Type σ ::=  recv τ; σ                   receive message type τ
                  | send τ; σ                   send message type τ
                  | choose {L:(σL) | R: (σR)}   choose sub-protocol
                  | offer {L: (σL) | R: (σR)}   offer sub-protocol
                  | label; σ                    label point
                  | goto i                      goto label
                  | ε                           end protocol
```
            
For example, this session type describes a ping server that sends an receives a ping in a loop, exiting on demand. The label/goto scheme uses de Bruign indices to locally encode label names as integers.

```
Label; offer { run: (send ping; recv ping; goto 0) | quit: (ε)}
```

The grammar, and this example, can be encoded in Rust like so:

```rust

struct Send<T, S>(PhantomData<(T, S)>);
struct Recv<T, S>(PhantomData<(T, S)>);
struct Offer<Left, Right>(PhantomData<(Left, Right)>);
struct Choose<Left, Right>(PhantomData<Left, Right>);
struct Label<S>(PhantomData<S>);
struct Goto<N>(PhantomData<N>);
struct Z;
struct S<N>(PhantomData<N>); // Peano encoding for natural numbers
struct Close;

struct Ping;
type PingServer = Label<Offer<Send<Ping, Recv<Ping, Goto<Z>>>, Close>>;

```
The runtime communication API uses the type-state concept as a channel whose type changes as the protocol advances. Initially, a `Chan` is created for the server and the client (the "dual" of the server).
Here's an example where the type annotations show the change.

```rust
fn example_ping_server() {
    let (c, _): (Chan<(), PingServer>, Chan<(), Dual<PingServer>>) = Chan::new();
    let mut c: Chan<(Offer<_,_>, ()), Offer<_,_>> = c.label;
    loop {
        c = match c.offer() {
            Branch::left(c) => {
                let c: Chan<_, Recv<_, _>> = c.send(Ping);
                let (c, Ping): (Chan<_, Goto<_>>, _) = c.recv();
                c.goto()
            },
            Branch::Right(c) => {
                c.close();
                return;
            }
        }
    }
}
```

Note that the `Chan` has two type arguments: an environment `Env` and a current action `Sigma`. The environment contains a list of session type generated by calls to `label`. When we `goto`, we loop up the corresponding type in the `Env` list and make that the type of the current channel.

> You might wonder how the method like `c.offer()` and `c.recv()` are actually implemented. Once the session type framework is established, they aren't very interesting - perform an operation, then transmute to the new type-state.
> For example, `recv`:
> ```rust
> impl<Env, T, S> Chan<Env, Recv<T, S>> where T: marker::Send + 'static {
>    pub fn recv(self) -> (Chan<Env, S>, T) {
>        unsafe {
>            let x = self.read();
>            (transmute(self), x)
>        }
>    }
>  }
> ```
> See the session-types crate for the full implementation if you are interested.

We are going to look at two type-level operations in this framework:

* Dual types: given a description of the protocol from one party's perspective (e.g `PingServer`) we can automatically generate a matching `PingClient` protocol. This is the `Dual<PingServer>` reference above.
* Labels and gotos: to implement `label/goto`, we need a notion of a dynamically-sized list of types that we can push, index and change.

## 2.1. Dual types
The idea of a dual session type is that if I am sending to you, you should be receiving from me. Similarly, if I offer to branch left or right, you should be choosing which branch to take. In Rust, the dual of `Send<i32, Close>` should be `Recv<i32, Close>`.

Dual session types are useful because they prevent errors. If you had to manually specify both halves of the protocol, you might accidentally mismatch one side.

We can write down an inductive procedure for generating a dual type as follows, using the notation  σ' to mean "the dual of  σ".

```
(send τ; σ)' = recv τ; σ'
(recv, τ; σ)' = send τ; σ'
(choose { L: (σL) | R: (σR) })' = offer { L: (σL') | R: (σR') }
(offer { L: (σL) | R: (σR) })' = offer { L: (σL') | R: (σR') }
(label; σ)' = label; σ'
(goto i)' = goto I
ε' = ε
```

In the context of type-level programming, σ' is an operator that takes as input a type, and produces a type. Like with MaxLevel, we encode that concept as a trait:

```rust
trait ComputeDual {
    type Output;
}

type Dual<S> = <S as ComputeDual>::Output;
```
Unlike before, `ComputeDual` only takes one argument `Self`, so it does not need additional parameters. Like before, we use an alias to simplify later usage of the trait.

The key idea is that each of the logical rules above cleanly translates into a corresponding trait implementation. First, the base cases:

```rust
impl ComputeDual for Close {
    type Output = Close;
}

impl<N> for ComputeDual for Goto<N> {
    type Output = Goto<N>;
}
```

To represent the inductive cases (e.g `Send`), we use a `where` clause to perform an inductive computation. For example:

```rust
impl<T, S> ComputeDual for Send<T, S> where S: ComputeDual {
    type Output = Recv<T, Dual<S>>;
}
```

Again, compare this code to the original rule:

```
(send τ; σ)' = recv τ; σ'
```

Usually, a `where` bound constraints a trait implementation, eg `ToString` for `Vec<T>` is only implemented where `T: ToString`. Here, we re-purpose the bound to perform a computation, i.e inductively getting the `Dual` of `S`.

> Note that trait bounds have the form `Type: Trait`, so we can't say `s: Dual` as `Dual` is a Type. We use `ComputeDual` in the trait bound, then `Dual<S>` when used as a Type.

As another example, here's the Dual rule for `Choose`:

```rust
impl<Left, Right> ComputeDual for Choose<Left, Right> where Left: ComputeDual, Right: ComputeDual {
    type Output = Offer<Dual<Left>, Dual<Right>>;
}
```

With these rules in hand, we can now easily specify our client type:

```rust
type PingServer = Label<Offe<..>>;
type PingClient = Dual<PingServer>;
```

That's it! We've successfully encoded dual session types as a type operator in Rust.

At this point, you may wonder - the translation from the pretty logic to the ugly traits involves a lot of syntax. Take a look at the type-operators crate for using a macro to automatically perform the translation.

## 2.2 Label and goto

Our final challenge is to implement the `label` and `goto` operators. We start with the following skeleton, needing to fill in the `?`:

```rust
impl<Env, S> Chan<Env, Label<S>> {
    // should push S onto Env
    pub fn label(self) -> Chan<?, S> {
        unsafe { transmute(self) }
    }
}

impl<Env, N> Chan<Env, Goto<N>> {
    // should get the Nth type in Env and drop the first N items From Env
    pub fn goto(self) -> Chan<?, ?> {
        unsafe { transmute(self) }
    }
}
```

We are going to encode `Env` as a list of session types. To do so, we need to resolve four questions:

1. How do we represent a list of types?
2. How do we `push` to a list of types?
3. How do we get the n-th type in a list of types?
4. How do we drop the first n types in a list of types?

Like in a functional programming language, we will use a linked-list nil/cons style to represent a type list.

```rust
type EmptyList = ();        // Or "Nil" if you prefer
type Push<L, T> = (T, L);   // or "Cons" if you prefer

type ExampleList = Push<Push<EmptyList, String>, bool>;
// ExampleList = (bool, (String, ()))
```

Then to get the n-th type in a list, we can use a type-level operator encoded as a trait, using a familiar pattern:

```rust
trait ComputeNth<N> { type Output; }
type Nth<L, N> = <L as ComputeNth<N>>::Output;
```

Think for a moment about the inductive definition of `Nth` as you might write it in Ocaml or Haskell. It probably looks something like this:

```
nth (τ,l) 0 = τ
nth (τ,l) i = nth l (i−1)
```

Just like with dual session types, we can straightfowardly encode these logical rules into trait implementations. However, because we are using a Peano encoding of the natural numbers, we'll tweak the second rule to look like this:

```
nth (τ,l) (i+1)=nth l i
```

Then the encoding of `Nth` into Rust traits become 1-to-1:

```rust
impl <T, L> ComputeNth<Z> for Push<T, L> {
    type Output T;
}

impl<T, L, N> ComputeNth<S<N>> for Push<T, L> where ComputeNth<N> {
    type Output = Nth<L, N>;
}
```

Similarly, we can create a function that drops the first `N` elements from a type list:

```rust
trait ComputeDropFirst<N> { type Output; }
type DropFirst<L, N> = <L as ComputeDropFirst<N>>::Output;
impl<L> ComputeDropFirst<Z> for L {
    type Output = L;
} 
impl <T, L, N> ComputeDropFirst<S<N>> for Push<T, L> where L: ComputeDropFirst<N> {
    type Output = DropFirst<L, N>;
}
```

With these type-level operators in hand, we can finish or label and goto implementations. Now, `label` is a simple `Push`:

```rust
impl<Env, S> Chan<Env, Label<S>> {
    pub fn label(self) -> Chan<Push<S, Env>, S> {
        unsafe { transmute(self) }
    }
}
```

The `goto` function is more complex. We need to both get the `Nth` element of an environment and drop the first `N` elements.

```rust
impl<Env, N> Chan<Env, Goto<N>> where Env: ComputeNth<N> + ComputeDropFirst<N> {
    pub fn goto(self) -> Chan<DropFirst<Env, N>, Nth<Env, N>> {
        unsafe { transmute(self) }
    }
}
```

Note that because we use `Env` with two type-level operators, we have to add both as bounds combined with `+`.

## 3. Traits as relations

The relationship between types, traits, and logic programming has been an enduring theme in the Rust community. "Lowering Rust traits to logic" and the ongoing efforts on chalk both show how resolving trait bounds is equivalent to executing a logic program.

In this note, I tried to go in the opposite direction - showing how domain-specific type-systems, whose rules are often written as logical relations, can be lowered into Rust traits. I think this is valuable because:

* Domain-specific type systems enable powerful compile-time checks on complex properties. Checking that communication protocols are followed, or that secure information is protected, usually requires external model checkers. Here, we just need Rust!
* Encoding logic rules into Rust traits in a practical way is not obvious at first glance. There are't many examples of this pattern to generalize from.

For example, if we took our type operators above, we could concisely encode their logic as a logic program (using Prolog-esque syntax):

```
% MaxLevel(Self, Other, Output)
MaxLevel(Low, Low, Low).
MaxLevel(Low, High, High).
MaxLevel(High, Low, High).
MaxLevel(High, High, High).

% Dual(Self, Output)
Dual(Close, Close).
Dual(Recv<T, S>, Send<T, S2>) :- Dual(S, S2).

% Nth(Self, N, Output)
Nth(X :: L, 0, X).
Nth(X :: L, I+1, X2) :- Nth(L, I, X2).
```

Seeing this pattern, we cna construct a general translation. A type parameter is a relation with Self as the first argument, `Output` as the last argument, and additional arguments in between. So a general relations `Rel(self, Arg1, .., ArgN, Output)` translate to:

```rust
trait ComputeRel<Arg1, ..., ArgN> { type Output: }
type Rel<Self, Arg1, ..., ArgN> = <Self as Rel<Arg1, ..., ArgN>>::Output;
```

An unconditional fact like `Rel(TSelf, T1, ... TN, Tout)` becomes an impl block without a where clause:

```rust
impl ComputeRel<T1, ...., Tn> for Tself { type Output = Tout; }
```
And a complex conditional fact like:

```
Rel(Tself, T1, .... Tn, Tout) :- OtherRel(Tself, TotherOut), Rel(TOtherOut, T1, ..., Tn, Tout).
```

Becomes an `impl` block with a `where` clause:

```rust
impl ComputeRel<T1, ...., TN> for TSelf where Tself: ComputeOtheRel, OtherReL<Tself>: ComputeRel<T1, ..., TN> {
    type Output = Rel<OtherRel<Tself>>;
}
```
In drawing this connection between traits and logic programs, I hope that you might find it easier to encode new domain-specific type systems in Rust. These examples also demonstrate that there's a lot of exciting future work in developing libraries and best practices for type-level programming!
