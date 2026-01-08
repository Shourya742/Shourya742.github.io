---
layout: "post"
title:  "Types over Strings"
date:   "2050-01-8 14:23:35 +0530"
categories: rust
--- 

Types are a useful tool to make sure software libraries work together. I expect an int, you give me a string, compiler raises an error. In my experience, types, interfaces and encapsulation work best when using block-box APIs: hash maps, regexes, HTTP requests. The programmer controls the top-level program, i.e. the types that orchestrate the individual pieces.

However, the Type Life tends to become harder when dealing with frameworks, or any kind of extensible architecture where the programmer is plugging components into a bigger system they don't control. For example, consider the venerable `EventTarget.addEventListener` from Javascript:

```rust
document.body.addEventListener("click", function(event) {
    console.log(event.clientX, event.clientY)
})
```

This is an extensible architecture in the sense that you don't control the DOM API nor can you control what events are registered to DOM nodes. That changes all the time. You can only hook in to events that browser provides.

Let's look at this function with two related questions: what is the type of `addEventListener`? And what kinds of errors can we make using this function?

The most basic type for `addEventListener(event, listener)` is that `event` is a string, and `Listener` is a function. A function for what to what? the input is an "event", whatever that means, and there is no output. Indeed, when `addEventListener` gets ported into a statically typed language like `ReasonML`, the type definition looks like:

```rust
external addEventListener: (string, Dom.event => unit) => unit;
```

Here's two ways we can mess this up:

* **Typo the event name**: as with all stringly-typed programming, we can write the event name incorrectly, like "clack" instead of "click". This error gets caught at runtime, usually when we register the handler.

* **Use the event object incorrectly**: while all events implement the base `Event interface`, each individual event has different fields. For example, "click" is a `MouseEvent` so it contains fields `clientX` and `clientY`. But what if I tried to access `clientX` on a `KeyboardEvent`? That error gets caught at runtime, when we execute the offending code path in the handler.

In this note, I'll show how you can avoid errors like these by designing type-safe extensible systems. We'll look at two examples: event handling and dependency injection. The code will use Rust, but the lessons apply to an function language.

## Type-safe event listeners

The basic idea in making type-safe extensible architecture is to replace strings with types. Any time you use a string as an identifier (e.g for an event, or a software component), use a type instead. Why?

* Types are better identifiers than strings. If you typo a type, the type checker can catch your mistake.

* Types can be associated with additional information. The string "click" means nothing to the type checker. But the type `ClickEvent` tels the typechecker about what fields the event has, and how event listeners should handle the event.

Here's an example of using a type-safe event API in Rust.

```rust
struct OnClick {
    mouse_x: i32,
    mouse_y: i32
}

let mut dispatcher = EventDispatcher::new();

dispatcher.add_event_listener(|event: &OnClick| {
    assert_eq!(event.mouse_x, 10);
    assert_eq!(event.mouse_y, 5);
});

dispatcher.trigger(&OnClick {
    mouse_x: 10,
    mouse_y: 5
})
```
In the snippet above, the event type is inferred from the type parameter `OnClick`, rather than using a string argument. This solves both of our errors:

* If we wrote &OnClack, the type checker will say `OnClack not found`.
* If we wrote `event.keyboard_input` instead of `event.mouse_x`, the typechecker will say `no field keyboard_input on type OnClick`.

Stupendous! But how it works? Here's the basic structure.

```rust
// Events must not contain pointers to things, for simplicity
trait Event: 'static {}

// An event listener is a function from an event to nil
trait EventListener<E: Event>  = FnMut(&E) -> () + 'static;

// An event dispatcher holds all the event listeners
struct EventDispatcher { /* .. */ }

impl EventDispatcher {
    fn add_event_dispatcher<E, F>(&mut self, f: F) 
    where
        E: Event
        F: EventListener<E> 
    {
        /* .. */
    }

    fn trigger<E>(&mut self, event: &E) 
    where
        E: Event
    {
        /* .. */
    }
}
```

In order to implement `EventDispatcher`, we need a data structure that can hold all the listeners. Given an event `E`, it should provide all the listeners for `E`. This raises two questions:

* How can we associate a listener with a type `E`?
* How can a single data structure hold listener for multiple events, which all have different types?

## Mapping types to values

We will briefly detour to make critical building block: the `TypeMap`. Based on our two requirements above, we will make a data structure that a> maps types to values and b> holds values of different types.

Rust has `std::any` for this purpose. `TypeId` allows us to get a unique, hashable identifier for each type. `Any` allows us to up-cast/down-cast objects at runtime. Hence, our `TypeMap` will map from `TypeId` to `Box<dyn Any>`

```rust
use std::collections::HashMap;
use std::any::{TypeId, Any};

struct TypeMap(HashMap<TypeId, Box<dyn Any>>);
```

To add an element to the map:
```rust
impl TypeMap {
    pub fn set<T: Any + 'static>(&mut self, t: T) {
        self.0.insert(TypeId::of::<T>(), Box::new(t))
    }
}
```
This means our map has one unique value for a given type. For example, if we use the `TypeMap` like this:

```rust
let mut map = TypeMap::new();
map.set::<i32>(1);
```

Then we insert a value `1` at the key `TypeId::of::<i32>()`. We can also implement `has` and `get` functions:

```rust
impl TypeMap {
    pub fn has<T: 'static + Any>(&self) -> bool {
        self.0.contains_key(&TypeId::of::<T>())
    }

    pub fn get_mut<T: 'static + Any>(&mut self) -> Option<&mut T> {
        self.0.get_mut(&TypeId::of::<T>()).map(|t| {
            t.downcast_mut::<T>().unwrap()
        })
    }
}
```

Look carefully at `get_mut`. The inner hash map returns a value of type `Box<dyn Any>`, which we can `downcast_mut` to become a value of type `&mut T`. This operation is guaranteed to not fail, because only values of type `T` are stored in the hashmap under the key for `T`.

## Finishing our event system

WIth the `TypeMap` in hand, we can finish our event system. For the `EventDispatcher`, the `TypeMap` will map from events to vector of listeners.

```rust
pub struct EventDispatcher(TypeMap);

type ListenerVec<E> = Vec<Box<dyn EventListener<E>>>;

impl EventDispatcher {
    pub fn add_event_listener<E, F>(&mut self, f: F)
    where
        E: Event,
        F: EventListener<E>,
    {
        if !self.0.has::<ListenerVec<E>>() {
            self.0.set::<Listener<Vec<E>>>(Vec::new())
        }
        let listener = self.0.get_mut::<ListenerVec<E>>().unwrap();
        listeners.push(Box::new(f))   
    }
}
```

When asked to register an event listener, we get the `ListenerVec` from the `TypeMap` and insert the new listener. Trigger is similarly simple:

```rust
impl EventDispatcher {
    pub fn trigger<E: Event>(&mut self, event: &E) {
        if let Some(listeners) = self.0.get_mut::<ListenerVec<E>>() {
            for callback in listeners {
                callback(event);
            }
        }
    }
}
```

That's it! Now we have a type-safe event system.

## Type-safe dependency injection

To avoid the impression that this pattern is specific to event listeners, I want to show another example of using types-over-string in extensible architectures. I started playing with specs recently, an entity-component system architecture written in Rust. Its `System` uses a dependency-injection-like pattern, so I want to distill that to a pedagogical example here.

Dependency injection systems are often rife with XML files, string keys, and other stringly-typed problems. DI makes for a good use case here.

### Motivation
The basic idea of dependency injection(DI) is that you have a component that depends on another, like a web server using a database. However, you don't want to hard-code a particular database constructor, and rather make it easy to swap in different database. For example:

```rust
trait Database {
    fn name(&self) -> &'static str;
}

struct MySQL;
impl Database for MySQl {
    fn name(&self) -> &'static str { "MySQL" }
}

struct Postgres:
impl Database for Postgres {
    fn name(&self) -> &'static str { "Postgres" }
}

struct WebServer { db: Box<dyn Database> }
impl WebServer {
    fn run(&self) {
        println!("Db name: {}", self.db.name());
    }
}
```

To implement DI, we need two things:

* We need a way to register a global `Database` at runtime to a particular instance, eg `MySQL` or `Postgres`.

* We need a way to describe a constructor for WebServer that fetches the registered `Database` instance.

## DI constructors

First, we'll define a trait `DIBuilder` that represents a constructor within our DI system.

```rust
trait DIBuilder {
    type Input;
    type Output:

    fn build(input: Self::Input) -> Self::Output;
}
```

The build method is a static method (doesn't take self as input). It just takes `Input` as input, and produces `Output` as output. The key idea is that because `Input` and `Output` are associated types, we can inspect them later on. We will need to find values for `Input` and to store `Output` in our DI manager.

We implement `DIBuilder` for each type in the system. The databases have no inputs, so their inputs is `()`. Their return type is `Box<dyn Database>`, meaning they are explicitly cast to a trait object so they can be used interchangeably.

```rust
impl DIBuilder for MySQL {
    type Input = ();
    type Output = Box<dyn Database>;
    fn build((): ()) -> Box<dyn Database> {
        Box::new(MySQl)
    }
}

impl DIBuilder for Postgres {
    type Input = ();
    type Output = Box<dyn Database>;

    fn build(():()) -> Box<dyn Database> {
        Box::new(Postgres)
    }
}

impl DIBuilder for WebServer {
    type Input = (Box<dyn Database>);
    type Output = WebServer;

    fn build((db,): Self::Input) -> WebServer {
        WebServer { 
            db
        }
    }
}
```

## DI Manager

Now that we know that dependency structure of our objects, we need a centralized manager to store the objects and fetch their dependencies.

```rust
struct DIManager(TypeMap);
```

In this `TypeMap`, we will store the constructed Objects. For example, once we make a `Box<dyn Database>`, then we will map `TypeId::of::<Box<dyn Database>>` to one of `Box<Postgres>` or Box<MySQl>.

```rust
impl DIManager {
    fn build<T: DIBuilder>(&mut self) -> Option<T::Output> {
        let input = /* get the inputs, somehow */;
        let obj = T::build(input);
        self.0.set::<T::Output>(obj);
        Some(obj)
    }
}
```
Ignoring how we fetch dependencies for now, this function calls the `DIBuilder::build` implementation for `T`, then stores the result in the `TypeMap`. This approach almost works, except not for Rust: ownership of `obj` is passed into `TypeMap` and the result of `build`.

And, intuitively, this makes sense. If a component like a database cursor needs to be shared across many downstream components, it needs some kind of access protection. Hence, we tweak our interface a bit to wrap everything in a `Arc` and `Mutex`.

```rust
type DIObj<T> = Arc<Mutex<T>>

impl DIManager {
    fn build<T: DIBuilder>(&mut self) -> Option<DIObj<T:: Output>> {
        let input = /* get the inputs, somehow*/;
        let obj = T::build(input);
        let syn_obj = Arc::new(Mutex::new(obj));
        self.0.set::<DIObj<T::Output>>(sync_obj.clone());
        Some(sync_obj)
    }
}
```

## DI dependencies

Finally, we need a way to implement the `let input` line in the `DIManager::build`. Given a Type `T: DIBuilder`, it has an associated type `T:Input` that represents the inputs needed to build it.

To simplify the problem, imagine `T::input = S` where `S:DIBuilder`. Then if `S` has already been built, e.g `T = WebServer` and `S = Box<dyn Database>`, we can fetch it directly from the typemap:

```rust
let input = self.0.get::<T::Input>().map(|obj| obj.clone())?;
```
However, in practice `T::Input` could be several dependencies. For example, if our server depends on a configuration, it might be:

```rust
impl DIBuilder for WebServer {
    type Input = (Box<dyn Database>, Box<dyn ServerConfig>);
    type Output = WebServer;

    fn build((db, config): Self::Input) -> WebServer {
        WebServer {
            db, config
        }
    }
}
```

Let's assume now `T::Input` is always a tuple `(S1, S2, ...)` of types where each type `S1: DIBuilder`. Ideally, we could write something like:

```rust
let input = (T::input).map(|S| {
    self.0.get::<S>().map(|obj| obj.clone()).unwrap()
});
```
But, alas, our language of expressions is not our language of types. Such a thing is the provenance of languages we can only `dream about`. Instead, we have to cleverly use traits to inductively define a way to extract inputs from the type. To start, we'll make a trait that gets an object of a particular type from the `DIManager`:

```rust
trait GetInput: Size {
    fn get_input(manage: &DIManager) -> Option<Self>;
}
```

For `DIObj<T>`, this means looking up the type in the `TypeMap`:

```rust
impl<T: 'static> GetInput for DIObj<T> {
    fn get_input(manager: &DIManager) -> Option<Self> {
        manager.0.get::<Self>().map(|obj| obj.clone())
    }
}
```

Then for tuples of `DIObj<T>`, we can make an inductive definition like so:

```rust
impl GetInput for () {
    fn get_input(_manager: &DIManager) -> Option<Self> {
        Some(())
    }
}

impl<S: GetInput, T: GetInput> GetInput for (S, T) {
    fn get_input(manager: &DIManager) -> Option<Self> {
        S::get_input(manager).and_then(|s| {
            T::get_input(manager).and_then(|t| {
                Some((s, t))
            })
        })
    }
}
```
Then we can modify our WebServer example to use an inductive structure:

```rust
impl DIBuilder for WebServer {
    type Input  = (DIObj<Box<dyn Database>>, (DIObject<Box<dyn ServerConfig>>, ()));
    type Output = WebServer;

    fn build((db, (config, ())): Self::input) -> WebServer {
        WebServer { db, config }
    }
}
```

And at last, we can implement `DIManager::build`:

```rust
impl DIManager {
    pub fn build<T: DIBuilder>(&mut self) -> Option<DIObj<T::Object>> {
        let input = T::Input::get_input(self)?;
        let obj = T::build(input);
        let sync_obj = Arc::new(Mutex::new(obj));
        self.0.set::<DIObj<T::Output>>(sync_obj.clone());
        Some(sync_obj)
    }
}
```
Now, the expression `T::Input::get_input(self)` will convert a tuple of types into a tuple of those types.

## Final API example

With the API complete, our example now looks like this:

```rust
let mut manager = DIManager::new();
manager.build::<MySQL>().unwrap();
let server = manager.build::<WebServer>().unwrap();
server.lock().unwrap().run();
```

We can construct a `WebServer` without explicitly passing in a `dyn Database` instance. When we use the server, we have to explicitly call .lock() now that it's wrapped in a mutex. And lo, our dependencies have been injected.


## Conclusion
Across these examples, there are a few main language level mechanisms that enable the high-level patterns:

* TypeID turns types into unique runtime identifiers. This allows us to have heterogeneous data structure like `TypeMap` while still getting compile time safety.

* Traits and associated types enable polymorphic functions like `EventDispatcher::add_Event_listener` and `DIManager::build` to statically ensure many valuable properties of the system. For example, an event listener can only be registered to an event that matches the type of its parameters.

* Polymorphic trait implementations enable type-level, like getting a value for each type in a arbitrary size tuple of types.

It seems that these mechanisms are important for type-safe extensible architectures.