+++
title = "`if err != nil` vs `Result<T, E>`: A Love Letter and a Breakup"

[taxonomies]
tags = ["Rust", "Go"]
+++

If there's one place I expected Rust to feel familiar, it was error handling. Go and Rust are the two mainstream languages that both looked at exceptions and said no thanks — errors are values, you check them, you move on. I've typed `if err != nil` enough times that my fingers do it without me. So naturally, my first instinct in Rust was to write the exact same pattern by hand, everywhere, for weeks, before I found out there was a much better way.

<!-- more -->

## The muscle memory transfers, at first

Go's version of this, you already know:

```go
func loadConfig(path string) (*Config, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, err
	}
	cfg, err := parse(data)
	if err != nil {
		return nil, err
	}
	return cfg, nil
}
```

Rust's `Result<T, E>` is the same idea with the compiler enforcing that you actually look at it:

```rust
fn load_config(path: &str) -> Result<Config, io::Error> {
    let data = fs::read(path);
    let data = match data {
        Ok(d) => d,
        Err(e) => return Err(e),
    };
    // ...
}
```

I got about two functions into writing Rust like this before it started to feel exactly as tedious as it looks. Every call needs its own `match`, every `match` needs its own early return. It's `if err != nil` with more ceremony, and I remember thinking: people really rave about this?

## Then I met `?`

Turns out I was writing it the hard way. The `?` operator does exactly what my hand-rolled `match` blocks were doing — check the result, return early on `Err`, unwrap on `Ok` — in one character:

```rust
fn load_config(path: &str) -> Result<Config, MyError> {
    let data = fs::read(path)?;
    let cfg = parse(&data)?;
    Ok(cfg)
}
```

This is the point where Rust error handling stopped feeling like a worse Go and started feeling like the thing Go proposals have been reaching for since forever — a way to propagate "this failed, stop and hand it up" without three lines of boilerplate per call. `?` even does automatic error conversion via the `From` trait, so if `fs::read` returns an `io::Error` and your function returns `MyError`, it'll convert on the way out as long as you've told it how (`impl From<io::Error> for MyError`). In Go I'd be writing `fmt.Errorf("reading config: %w", err)` by hand at every layer boundary; here the conversion is defined once and just happens.

That's the love letter part.

## `Option<T>` and the nil I don't miss

The other half of this is `Option<T>`, and it's where I stopped missing Go's zero values almost immediately. Every Go developer has been bitten by this at least once:

```go
var cfg *Config
fmt.Println(cfg.Name) // panic: nil pointer dereference
```

Nothing in the type system stops you from doing this. `*Config` claims to be a config, but it might just be a landmine. Rust doesn't let a value pretend to exist when it might not — if something might be absent, its type says so, out loud:

```rust
let cfg: Option<Config> = find_config();
match cfg {
    Some(c) => println!("{}", c.name),
    None => println!("no config found"),
}
```

You cannot accidentally call `.name` on a `None`. The compiler won't even let you get at the inner `Config` without acknowledging the `None` case exists somewhere — either a `match`, an `if let`, or a method like `.unwrap_or_default()` that makes the fallback explicit. It's the same shape as `Result`, and once that clicked, half of Rust's standard library stopped looking like a pile of special-cased types and started looking like the same one or two ideas applied consistently everywhere.

## The breakup part

Here's the thing nobody warns you about: `.unwrap()` is right there, and it is *so easy to reach for*. In Go, panicking on an error you could have checked feels wrong enough that you rarely do it — the idiomatic path and the lazy path are the same path. In Rust, the lazy path is `.unwrap()`, and the idiomatic path is an honest `match` or a `?`, and those are visibly different amounts of effort:

```rust
let cfg = load_config("app.toml").unwrap(); // fine for a weekend project...
```

Every early Rust program I wrote was riddled with these, because when you're fighting the borrow checker with one hand you don't have a second hand free to write proper error handling too. The difference from Go's nil panics is that this one is opt-in — I put the landmine there myself, which somehow makes it worse.

And then there's picking a story for custom errors. Go has basically one idiom: `errors.New`, `fmt.Errorf("%w", err)`, and `errors.Is`/`errors.As` if you need to inspect the chain. Rust hands you `enum MyError { Io(io::Error), Parse(ParseError) }` plus an `impl std::error::Error`, and then the ecosystem immediately splits into "just use `anyhow` for applications" and "use `thiserror` for libraries," and as a newcomer nobody tells you which one you're supposed to reach for until you've already picked wrong once.

## What I still don't love

- **`.unwrap()` is too easy to lean on.** It's a panic wearing a method call's clothes, and early on I couldn't tell the difference between "I'm prototyping" and "I'm building a landmine."
- **The `anyhow` vs `thiserror` decision.** Go gives you one way to do this. Rust gives you two good ones and expects you to know why.
- **`?` is magic until it isn't.** The automatic `From` conversion is great right up until you have three error types in scope and the compiler can't figure out which conversion you meant, and the resulting message is a wall of trait bound text.

Next up: Go doesn't really have enums, just `iota` and a promise you won't misuse it. Rust's `enum` is where I found out what I'd actually been missing.
