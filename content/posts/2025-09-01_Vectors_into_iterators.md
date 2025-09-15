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
