+++
title = "Rust Vectors into Iterators"
+++

Converting a Rust Vector into an Iterator


<!-- more -->


Vectors can also be converted into iterators just like sets. The syntax to converts a vector into an iterator is:

```rust
let v_iter = v.into_iter();

```


Example:

```rust
fn main() {
	let v = Vec::from([2,4,8,10]);
	
	let v_iter = v.into_iter();
	
	for item in v_iter {
		println!("{}", item);
	}
} 
```


An iterator can be converted back into a vector using the `collect()` method, we need to specify the type of the vector we want to convert it into.

```rust
let v: Vec<i32> = my_iter.collect(); // turns the iterator into a vector
let v: HashSet<i32> = my_other_iter.collect(); // turns iter into a set
```