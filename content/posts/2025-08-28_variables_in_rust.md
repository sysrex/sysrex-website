+++
title = "Variables in Rust"
description = "A partial excerpt of Amos Wenger's A half-hour to learn Rust, on let, shadowing, and the underscore. The original is the piece to read."

[taxonomies]
tags = ["Rust"]
+++

These notes are a partial excerpt of Amos Wenger's [A half-hour to learn Rust](https://fasterthanli.me/articles/a-half-hour-to-learn-rust), kept while I was learning `let`. Read the original. I'm leaving this page up so the URL still resolves, until I replace it with a note of my own.

<!-- more -->

## Variable bindings

### The `let` keyword

`let` introduces a variable binding:

```rust
let x; // declare "x"
x = 42; // assign 42 to "x"
```

This can also be written as a single line:

```rust
let x = 42;
```

### Type annotation
You can specify the variable’s type explicitly with :, that’s a type annotation:

```rust
let x: i32; // `i32` is a signed 32-bit integer
x = 42;
```



The same binding, with the type on one line:

```rust
// there's i8, i16, i32, i64, i128
//    also u8, u16, u32, u64, u128 for unsigned
let x: i32 = 42;
```

#### Uninitialized variables
If you declare a name and initialize it later, the compiler will prevent you from using it before it’s initialized.

```rust
let x;
foobar(x); // error: borrow of possibly-uninitialized variable: `x`
x = 42;
```

However, doing this is completely fine:

```rust
let x;
x = 42;
foobar(x); // the type of `x` will be inferred from here
```

#### Throwing values away
The underscore _ is a special name - or rather, a “lack of name”. It basically means to throw away something:


```rust
// this does *nothing* because 42 is a constant
let _ = 42;
```


```rust
// this calls `get_thing` but throws away its result
let _ = get_thing();
```
Names that start with an underscore are regular names, it’s just that the compiler won’t warn about them being unused:

```rust
// we may use `_x` eventually, but our code is a work-in-progress
// and we just wanted to get rid of a compiler warning for now.
let _x = 42;
```

#### Shadowing bindings
Separate bindings with the same name can be introduced - you can shadow a variable binding:

```rust
let x = 13;
let x = x + 3;
// using `x` after that line only refers to the second `x`,
//
// although the first `x` still exists (it'll be dropped
// when going out of scope), you can no longer refer to it.
```
