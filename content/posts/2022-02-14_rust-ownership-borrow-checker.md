+++
title = "Rust Ownership Made Me Feel Like a Junior Dev Again"

[taxonomies]
tags = ["Rust", "Go"]
+++

I've been writing Go for years. I know what a pointer is, I know what a data race is, and I've made my peace with `go test -race` catching the ones I missed. So when I sat down to write my first real Rust program, I expected the syntax to be annoying and the rest to be familiar. Ten minutes in, the compiler told me I couldn't use a variable I had clearly just created, and I genuinely didn't understand why.

<!-- more -->

## The program that broke me

Here's roughly what I was trying to do — nothing exotic, just the kind of thing you write without thinking in Go:

```go
func main() {
	config := loadConfig()
	startServer(config)
	logStartup(config)
}
```

Load a thing, pass it to a function, pass it to another function. In Go this is a non-event. `config` is a value (or a pointer to one), and you can hand it to as many functions as you want. The garbage collector will figure out when it's actually done with.

I wrote what felt like the equivalent in Rust:

```rust
fn main() {
    let config = load_config();
    start_server(config);
    log_startup(config);
}
```

And got this:

```
error[E0382]: use of moved value: `config`
  --> src/main.rs:4:17
   |
2  |     let config = load_config();
   |         ------ move occurs because `config` has type `Config`, which does not implement the `Copy` trait
3  |     start_server(config);
   |                  ------ value moved here
4  |     log_startup(config);
   |                 ^^^^^^ value used here after move
```

My first reaction was that I'd found a compiler bug. My second reaction, after re-reading it three times, was: wait, does calling a function *destroy* my variable?

## Ownership, the short version

Sort of, yes. In Rust, every value has exactly one owner, and when you pass that value into a function by its plain type (not a reference), ownership moves with it. `start_server(config)` doesn't borrow `config` — it *takes* it. Once that line runs, `config` isn't a stale-but-usable variable anymore, it's gone, and the compiler will refuse to let you touch it again. That's not a runtime check either — it's caught at compile time, before the program ever runs.

Go doesn't have this concept because Go doesn't need it: the GC keeps values alive as long as anything references them, so you can pass the same slice or pointer to five different functions and never think about who "owns" it. Rust has no GC, so the compiler has to know, statically, exactly when a value's memory can be freed — and it does that by insisting there's always exactly one owner responsible for it.

The fix, once I understood what was happening, was almost insultingly simple:

```rust
fn main() {
    let config = load_config();
    start_server(&config);
    log_startup(&config);
}
```

`&config` borrows the value instead of taking it. Ownership stays in `main`, and both functions just get read access for the duration of the call. This is the part that actually maps to something I already knew from Go — it's a lot like the difference between passing a value and passing a pointer, except the compiler is now checking that I'm not lying about how I'm using it.

## Borrowing has rules, and they're stricter than I expected

Where this stopped feeling like "pointers with extra steps" was the borrowing rules themselves. At any given time, you can have either:

- any number of immutable references (`&T`), or
- exactly one mutable reference (`&mut T`)

...but never both at once. Try to take a mutable borrow while an immutable one is still alive, and the compiler stops you cold:

```rust
let mut scores = vec![1, 2, 3];
let first = &scores[0];
scores.push(4); // error: cannot borrow `scores` as mutable
println!("{}", first);
```

In Go, this exact pattern — reading from a slice while another goroutine appends to it — is precisely the kind of thing that compiles fine, runs fine in your dev environment, and then blows up under load or gets flagged by `-race` if you're lucky enough to have a test that exercises it. Rust just... doesn't let you write it. Not "warns you," not "catches it at runtime" — the code does not compile.

That reframe took me a while to actually feel. My gut reaction to fighting the borrow checker for an hour over what felt like a trivial function was that the language was getting in my way. But it's not really a new restriction — it's the same rule I was already supposed to be following in Go (don't mutate something while someone else might be reading it), except in Go that rule lives in my head and in code review, and in Rust it lives in the compiler.

## What I still don't love

I'm not going to pretend this all clicked and now I'm enlightened. A few things still genuinely slow me down:

- **`.clone()` as an escape hatch.** When I don't understand why the borrow checker is upset, my first instinct is still to slap `.clone()` on something and move on. It works, but it's the Rust equivalent of wrapping something in a mutex because you don't want to think about it — a crutch, not a fix.
- **Structs that reference other structs.** The moment you want one struct to hold a reference into another, you're into lifetime annotations (`'a`), and that's a whole separate fight I haven't fully made peace with yet.
- **The error messages are good, but they assume you already know the vocabulary.** "Cannot borrow as mutable because it is also borrowed as immutable" makes total sense once you know what borrowing means. It reads like Greek the first dozen times.

Next up: what happens when I bring my `if err != nil` muscle memory to `Result<T, E>` — which, unlike ownership, actually felt like coming home.
