+++
title = "Why Ethereum Execution Clients Are Being Rewritten in Rust"

[taxonomies]
tags = ["Rust", "Ethereum"]
+++

If you've spent any time around L2 rollups — including the op-stack setup
covered in an earlier post here — you've run into "execution clients":
the piece of software responsible for actually running transactions and
maintaining state. Go has dominated this space for years via Geth. More
recently, a wave of new clients (Reth being the most visible) have been
written in Rust instead. The reasons why are a good case study in what
Rust is actually for.

<!-- more -->

## What an execution client has to do

Strip away the consensus layer and the networking, and an execution
client's core job is deceptively simple to describe: given a block of
transactions, execute them against the current state (account balances,
contract storage, nonces) and produce the new state root. In practice this
means:

- Running the EVM itself, transaction by transaction.
- Reading and writing a *lot* of small key-value pairs — every storage
  slot touched by every transaction — against an on-disk database.
- Maintaining a Merkle Patricia Trie (or a variant of one) so that the
  entire state can be summarized by a single 32-byte root hash.
- Doing all of this fast enough to keep up with mainnet block times, and
  ideally fast enough to replay historical blocks quickly for syncing.

The bottleneck in practice is rarely EVM execution itself — it's I/O.
State access patterns are punishing: lots of small, semi-random reads and
writes against a database that can grow into the hundreds of gigabytes.

## Where Go starts to hurt

Go is a perfectly reasonable language for building network services, and
Geth's dominance for years shows it's entirely workable for an execution
client too. But at the I/O-bound, latency-sensitive scale these clients
operate at, a few things start to bite:

- **GC pauses**: Go's garbage collector is quite good, but it's still a
  tracing GC running concurrently with your program, and under heavy
  allocation pressure (which state-heavy workloads produce a lot of) it
  can introduce latency spikes that are hard to fully eliminate.
- **Less control over memory layout**: squeezing out the last bit of
  performance from a hot path — arranging structs to be cache-friendly,
  avoiding pointer-chasing, controlling exactly when and how memory is
  allocated — is harder when the language wasn't designed to give you that
  level of control.

## What Rust brings to this specific problem

Rust's pitch for this workload maps closely onto what execution clients
actually need:

- **No garbage collector** — memory is managed via ownership and
  lifetimes, checked at compile time, so there's no GC pause to worry
  about at all. For a process where consistent low latency under load
  matters, removing an entire class of latency spike is valuable.
- **Zero-cost abstractions** — you can write reasonably high-level code
  and still get performance close to hand-tuned C, which matters when the
  hot path is executed millions of times per sync.
- **Fine-grained control over memory layout and allocation**, which
  matters a lot for a workload dominated by small, frequent key-value
  operations against an embedded database.
- **A mature systems-programming ecosystem** — crates for cryptography,
  networking (libp2p has strong Rust support), and embedded databases
  (Reth uses MDBX, an embedded key-value store) that make it practical to
  build this kind of software without reinventing everything from scratch.

## The trade-off

None of this is free. Rust's compile times are longer, its learning curve
is steeper (the borrow checker takes real time to get comfortable with),
and Go's simplicity has genuine value for a large open-source project with
many contributors of varying experience levels. The bet that Reth and
similar projects are making is that for *this specific* workload —
sustained, latency-sensitive, I/O-heavy, running as critical infrastructure
that L2s and node operators depend on — the performance and predictability
gains are worth the steeper development cost.

It's also worth noting this isn't purely academic for the rollup world:
several L2 execution clients are themselves forks or adaptations of
mainnet execution clients (op-geth being the most visible example, with
Rust-based equivalents following the same pattern as Reth matures). The
execution client layer and the rollup layer are more entangled than they
might first appear — which is part of why performance work at the base
execution client level tends to ripple outward into the L2 ecosystem built
on top of it.
