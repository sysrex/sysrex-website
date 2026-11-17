+++
title = "Rust Vectors into Iterators"
description = "into_iter consumes a Vec. collect, with a type annotation, turns the iterator into a Vec or another collection."

[taxonomies]
tags = ["Rust"]
+++

`into_iter` turns a vector into an iterator by taking ownership of it. After that call the vector can't be used again. `iter` borrows the elements, and `iter_mut` borrows them mutably. This note only shows the consuming form.

<!-- more -->

```rust
let v_iter = v.into_iter();
```

```rust
fn main() {
    let v = Vec::from([2, 4, 8, 10]);
    let v_iter = v.into_iter();

    for item in v_iter {
        println!("{}", item);
    }
}
```

`collect` builds a collection from an iterator. The type annotation is what decides which one:

```rust
use std::collections::HashSet;

let v: Vec<i32> = my_iter.collect();
let set: HashSet<i32> = my_other_iter.collect();
```
