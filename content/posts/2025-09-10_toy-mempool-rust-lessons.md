+++
title = "I Built a Toy Mempool to Find Out If Any of the Rust Actually Stuck"

[taxonomies]
tags = ["Rust", "Go", "Ethereum"]
+++

Four articles ago I couldn't get a config struct past the borrow checker. Since then I've written about `Result<T, E>`, fought the orphan rule over a trait I didn't own, and finally understood why `Arc<Mutex<T>>` isn't just `sync.Mutex` with extra steps. All of that was still theory, though — small examples built to make one point each. I wanted to know if it would hold up in something with more than one moving part, so I gave myself an afternoon to build a toy version of the piece of an [execution client](/posts/ethereum-execution-clients-rust/) I already understood from the Go side: the mempool, the queue of pending transactions waiting to get into a block.

<!-- more -->

## Why a mempool, specifically

A mempool is small enough to build in an evening and still touches everything the series covered: transactions come in different shapes (enums and traits), they need to be validated before they're accepted (`Result`, not exceptions), and in a real client they arrive concurrently from dozens of peers at once (ownership, `Arc`, `Mutex`). If the four articles' worth of pain was going to pay for itself anywhere, it would be here.

## The shape of a transaction

Ethereum has had two transaction formats since EIP-1559 — legacy, with a flat gas price, and the newer one with a priority fee and a cap. In Go I'd have reached for a single struct with a bunch of optional fields and a type tag to say which ones mattered. Rust's enum makes that distinction load-bearing instead of a convention:

```rust
#[derive(Debug, Clone)]
enum TxKind {
    Legacy { gas_price: u64 },
    Eip1559 { max_fee_per_gas: u64, max_priority_fee_per_gas: u64 },
}

#[derive(Debug, Clone)]
struct Transaction {
    from: [u8; 20],
    nonce: u64,
    kind: TxKind,
}
```

Computing what a transaction actually pays differs by kind, which is exactly what traits are for:

```rust
trait GasPricing {
    fn effective_gas_price(&self, base_fee: u64) -> u64;

    fn is_underpriced(&self, base_fee: u64) -> bool {
        self.effective_gas_price(base_fee) < base_fee
    }
}

impl GasPricing for Transaction {
    fn effective_gas_price(&self, base_fee: u64) -> u64 {
        match self.kind {
            TxKind::Legacy { gas_price } => gas_price,
            TxKind::Eip1559 { max_fee_per_gas, max_priority_fee_per_gas } => {
                base_fee.saturating_add(max_priority_fee_per_gas).min(max_fee_per_gas)
            }
        }
    }
}
```

`is_underpriced` gets a default implementation for free, and the `match` on `TxKind` is exhaustive — if EIP-4844's blob transactions ever get added to this enum, the compiler stops the build everywhere I forgot to handle the new variant. In Go, the equivalent would be a `switch` on a type-tag field that fails silently at runtime if someone adds a case and forgets a callsite. That's article three's whole argument, showing up on the first page of actual code.

## Rejecting transactions without an if-err-chain

A transaction can be rejected for a handful of reasons — stale nonce, underpriced — and I wanted the caller to be unable to ignore which one:

```rust
#[derive(Debug)]
enum MempoolError {
    NonceTooLow { expected: u64, got: u64 },
    Underpriced,
}

fn validate(tx: &Transaction, expected_nonce: u64, base_fee: u64) -> Result<(), MempoolError> {
    if tx.nonce < expected_nonce {
        return Err(MempoolError::NonceTooLow { expected: expected_nonce, got: tx.nonce });
    }
    if tx.is_underpriced(base_fee) {
        return Err(MempoolError::Underpriced);
    }
    Ok(())
}
```

Nothing here is new after article two, but it's the first time it felt like the natural tool instead of the thing I was learning. `submit` just chains it with `?` and moves on:

```rust
fn submit(pool: &SharedPool, tx: Transaction, expected_nonce: u64, base_fee: u64) -> Result<(), MempoolError> {
    validate(&tx, expected_nonce, base_fee)?;
    let mut pending = pool.lock().unwrap();
    pending.entry(tx.from).or_insert_with(Vec::new).push(tx);
    Ok(())
}
```

## Letting more than one peer submit at once

This is the part a real mempool can't avoid: transactions arrive from many peers concurrently, and they all want to touch the same pool. I simulated that with a handful of threads standing in for peers:

```rust
type SharedPool = Arc<Mutex<HashMap<[u8; 20], Vec<Transaction>>>>;

fn main() {
    let pool: SharedPool = Arc::new(Mutex::new(HashMap::new()));
    let mut handles = vec![];

    for peer_id in 0..4 {
        let pool = Arc::clone(&pool);
        handles.push(thread::spawn(move || {
            let tx = fake_tx_from_peer(peer_id);
            if let Err(e) = submit(&pool, tx, 0, 10) {
                eprintln!("peer {peer_id} rejected: {e:?}");
            }
        }));
    }

    for h in handles {
        h.join().unwrap();
    }
}
```

`Arc::clone` per thread, a `Mutex` around the map, done — this is exactly the pattern from article four. But the moment that actually convinced me was a mistake I made trying to add a convenience function to peek at a peer's pending transactions:

```rust
fn peek(pool: &SharedPool, addr: [u8; 20]) -> &Transaction {
    let pending = pool.lock().unwrap();
    &pending[&addr][0] // error: cannot return value referencing temporary value
}
```

In Go I'd have written the equivalent without a second thought — grab the lock, return a pointer into the map, unlock somewhere and hope nobody's still holding that pointer when the next write happens. Rust won't compile it: the `MutexGuard` is dropped at the end of the function, so the reference it hands out can't outlive it. The fix is to clone the transaction out while the lock is held, which is more copying than I wanted, but it's copying instead of the exact class of bug — reading through a pointer to memory another goroutine is mid-mutation on — that this whole series has been about avoiding.

## What still wasn't free

- **Every peer clones an `Arc` before it can touch the pool.** It's one line, but coming from Go where a captured variable in a closure just works, it's a line I have to remember every single time.
- **Lock granularity.** One `Mutex` around the whole map means every submission serializes against every other one, which real mempools obviously can't afford — Reth shards and uses lock-free structures where it matters. My toy version traded throughput for not having to think about it, which was the right trade for an evening project and would be the wrong one for a real client.
- **The validation errors are still just an enum I match on by hand.** A real mempool also has to decide what to evict when it's full, which is a policy question, not a type-system one — Rust doesn't have an opinion here, and neither do I yet.

## Did it stick

Mostly. I didn't reach for `.clone()` out of confusion once, the exhaustive `match` on `TxKind` caught a case I'd genuinely forgotten while writing this, and the `peek` mistake above is a bug I would have shipped in Go and caught three weeks later from a panic in production, not from the compiler while writing the function. None of it felt like fighting anymore — it felt like the language agreeing with something I already believed about concurrent code and just enforcing it earlier than I would have. That's the whole pitch behind [why projects like Reth exist at all](/posts/ethereum-execution-clients-rust/): the pressure I felt writing a hundred lines of toy mempool is the same pressure, at a much larger scale, that's rewriting Ethereum's execution layer out from under Go.
