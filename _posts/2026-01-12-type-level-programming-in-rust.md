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

```