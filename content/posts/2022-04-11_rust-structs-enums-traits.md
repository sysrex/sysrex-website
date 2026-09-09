+++
title = "Go Has One Enum-Shaped Hole and Rust Fills It Three Times Over"

[taxonomies]
tags = ["Rust", "Go"]
+++

Go doesn't have enums. It has `iota` and a gentleman's agreement that you won't pass it a number that doesn't mean anything. I'd made my peace with that years ago — it's just how Go is. Then I wrote my first Rust `enum` with actual data attached to its variants, and realized I'd been quietly working around a hole in the type system for my entire Go career without noticing it was a hole.

<!-- more -->

## What I was used to calling an enum

Here's the Go pattern everyone reaches for:

```go
type State int

const (
	Pending State = iota
	Running
	Done
)
```

It works, and it reads fine, but `State` is just an `int` wearing a costume. Nothing stops you from writing `State(99)`, and nothing stops a `switch` from silently falling through a case you forgot to handle:

```go
switch s {
case Pending:
	fmt.Println("waiting")
case Running:
	fmt.Println("in progress")
// forgot Done — compiles fine, silently does nothing
}
```

I'd never really thought of this as a limitation. It's just what a Go enum is.

## Rust's enum actually carries data

The first surprise is that Rust enum variants aren't just labels — each one can hold its own, different data:

```rust
enum Event {
    Connected(SocketAddr),
    Disconnected { reason: String },
    Message(Vec<u8>),
}
```

`Connected` carries an address. `Disconnected` carries a named field. `Message` carries a payload. In Go this would be three separate structs and probably an interface to tie them together, plus a type switch to figure out which one you actually got. Here it's one type, and `match` forces you to handle every variant:

```rust
match event {
    Event::Connected(addr) => println!("connected: {addr}"),
    Event::Disconnected { reason } => println!("dropped: {reason}"),
    Event::Message(bytes) => println!("{} bytes", bytes.len()),
}
```

Leave one variant out and the compiler refuses to build — not a lint warning, a hard error. The first time I deleted a match arm by accident and got `error[E0004]: non-exhaustive patterns`, I felt a specific kind of relief I didn't know I'd been missing since the last time a Go `switch` silently ate a case in production.

## The moment `Option` and `Result` stopped feeling like special syntax

This is where the previous article's `Option<T>` and `Result<T, E>` clicked into place for me. They're not built-in magic — they're just enums, defined in the standard library the same way I'd define my own:

```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Once I saw that, I stopped treating them as two special cases to memorize and started seeing them as the same one mechanism — a type that says "it's one of these specific shapes, and the compiler will make you handle all of them" — applied consistently across the entire language. Go doesn't have an equivalent primitive, so every library invents its own way to represent "this or that" (a pointer that might be nil, an interface with a type switch, a bool alongside a value), and none of them get the exhaustiveness check for free.

## Traits are not interfaces, even though they look like it

Structs themselves felt almost boring after all that — fields, methods, not far from a Go struct. The `#[derive(Debug, Clone)]` attribute standing in for what Go gets from an implicit `String()` method or manual `DeepCopy` felt like a nice shortcut, not a new concept.

Traits are where I got overconfident and then got corrected. A Go interface is satisfied structurally — if your type has the right methods, it implements the interface, full stop, no declaration required:

```go
type Stringer interface {
	String() string
}

// satisfies Stringer just by having the method — no `implements` keyword anywhere
type Point struct{ X, Y int }
func (p Point) String() string { return fmt.Sprintf("(%d,%d)", p.X, p.Y) }
```

I assumed Rust traits worked the same way, structurally. They don't. You have to say so:

```rust
trait Describe {
    fn describe(&self) -> String;
}

impl Describe for Point {
    fn describe(&self) -> String {
        format!("({}, {})", self.x, self.y)
    }
}
```

No `impl` block, no trait, even if the method already exists on the type with the exact right signature. My first reaction was that this was pure ceremony. My second reaction, after accidentally satisfying a Go interface I never meant to implement (the classic "oh no, this type now silently matches `io.Writer`" surprise), was that maybe requiring the declaration up front isn't ceremony, it's the same guardrail as exhaustive `match` — the compiler making sure nothing happens by accident.

Traits also do a couple of things Go interfaces can't: default method bodies (implement once, override only where you need to), and associated types that tie a second type to the implementation. Go would need embedding tricks and generics gymnastics to fake either of those.

## What I still don't love

- **The orphan rule.** You can't `impl` a foreign trait for a foreign type — I hit this trying to add a helper trait implementation on `Vec<T>` from a std library type I didn't own, and had to learn about wrapper (newtype) structs just to work around it.
- **Generics vs `dyn Trait`.** Go has one way to be polymorphic — an interface value. Rust makes you choose up front between compile-time generics (`fn process<T: Describe>(item: T)`) and runtime trait objects (`fn process(item: &dyn Describe)`), and as a newcomer I didn't yet have the instinct for which one a given situation wanted.
- **Trait bound errors are a paragraph long.** When a generic function's constraints don't line up, the error message is honest and complete and about fifteen lines of `where` clauses I have to read twice.

Next up: concurrency, which is the actual reason I picked up Rust in the first place — and where goroutines both prepared me and completely failed to prepare me.
