+++
title = "Goroutines Spoiled Me, and Rust Un-Spoiled Me"

[taxonomies]
tags = ["Rust", "Go"]
+++

Concurrency is the actual reason I picked up Rust. Go already sold me on the idea that concurrency should be a first-class, unscary part of a language — `go func()` and a channel, done. So when people said Rust was "fearless concurrency," I expected either a nicer version of what I already had, or marketing. It turned out to be neither. It's the same promise Go makes, kept in a completely different way, and it cost me more up front than I expected.

<!-- more -->

## The part that felt like coming home

Spawning work and communicating over a channel looks almost suspiciously familiar:

```go
ch := make(chan int)
go func() {
	ch <- compute()
}()
result := <-ch
```

```rust
let (tx, rx) = mpsc::channel();
thread::spawn(move || {
    tx.send(compute()).unwrap();
});
let result = rx.recv().unwrap();
```

Same shape, same idea: don't share memory to communicate, communicate to share memory. I remember being relieved that this part, at least, wasn't going to be a fight.

## The part that wasn't: there's no goroutine

Here's what actually is different, and it took me embarrassingly long to notice: `thread::spawn` gives you a real OS thread. Not a goroutine. Go's runtime multiplexes potentially hundreds of thousands of goroutines onto a handful of OS threads for free, and I had genuinely never had to think about the cost of `go func()` because the runtime made sure I didn't need to. Spawn ten thousand OS threads in Rust the naive way and you will feel it.

If you want something goroutine-shaped — lightweight, cheap to spawn by the thousands — you don't get it by default. You reach for `async`/`.await` and a runtime like Tokio, and that is a real decision with real weight, not a language feature that's just always on. Go never makes you choose a runtime. Rust makes you choose one, understand what a `Future` is, and think about pinning before you're done. That's not a skill issue on my part — it's genuinely more conceptual surface area than `go func()`, and I'd rather say that plainly than pretend it clicked instantly.

## Where the "fearless" part actually shows up

The payoff, once I got past the setup cost, is real. In Go, sharing a value across goroutines without a mutex compiles fine, runs fine in dev, and is a landmine that `go test -race` might catch — if you're lucky enough to have a test that actually exercises the race:

```go
counter := 0
for i := 0; i < 100; i++ {
	go func() { counter++ }() // data race, compiles anyway
}
```

Rust's `Send` and `Sync` traits turn that same mistake into a compile error instead of a race detector footnote. Try to share something across threads that isn't safe to share, and you don't get a runtime warning — you get a wall that stops the build:

```rust
let counter = Rc::new(RefCell::new(0));
thread::spawn(move || {
    *counter.borrow_mut() += 1; // error: `Rc<RefCell<i32>>` cannot be sent between threads safely
});
```

`Rc` and `RefCell` are the single-threaded tools — cheap reference counting and interior mutability with no locking, because on one thread you don't need any. The compiler knows that, and refuses to let them cross a thread boundary. The fix is switching to the thread-safe versions:

```rust
let counter = Arc::new(Mutex::new(0));
let c = Arc::clone(&counter);
thread::spawn(move || {
    *c.lock().unwrap() += 1;
});
```

`Arc` instead of `Rc`, `Mutex` instead of `RefCell`. Two extra letters and an explicit lock, and in exchange the class of bug I've spent actual on-call hours chasing in Go — the one that only shows up under production load, three weeks after the PR merged — simply doesn't compile. That's the first time in this whole series where the earlier pain, all of it, ownership and borrowing and the fights over `.clone()`, felt like it had obviously been worth it. `Send` and `Sync` aren't new rules bolted onto threading; they're the ownership rules from article one, extended to say who's allowed to hand a value to another thread at all.

## What actually won me over

Not any single feature, honestly. It was the moment I realized the borrow checker fight from four articles ago and the compile error above are the same mechanism doing two different jobs. Go gave me concurrency that's cheap and easy to start and occasionally, quietly, wrong. Rust gave me concurrency that costs more up front — a runtime to choose, a `Send` bound to satisfy, a `Mutex` to reach for instead of a `RefCell` — in exchange for the compiler ruling out an entire category of bug before the code even runs.

I still reach for Go when I want to ship something fast and the stakes are "restart the pod if it falls over." I reach for Rust now when the stakes are higher than that, and I don't want to find out about the race condition from a pager alert. Four articles ago I thought the borrow checker was the whole story. Turns out it was just the first receipt for something I'd end up cashing in every article after.
