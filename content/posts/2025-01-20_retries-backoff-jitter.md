+++
title = "Retries, Backoff, and Jitter: Why Naive Retry Logic Makes Outages Worse"

[taxonomies]
tags = ["Distributed Systems"]
+++

"Just retry the request" is one of the most reasonable-sounding pieces of
advice in distributed systems, and one of the easiest to get badly wrong.
Done carelessly, retries don't make your system more resilient — they turn
a small, recoverable blip into a full outage.

<!-- more -->

## The retry storm problem

Imagine a downstream service gets slow — maybe a database is under load,
maybe a dependency is having a bad day. Requests start timing out. If every
client immediately retries on timeout, you've just doubled the load on a
service that was already struggling. If those retries also fail, and the
clients retry again immediately, load keeps multiplying. This is a retry
storm, and it's a common way a minor slowdown turns into a total outage:
the retries themselves become the dominant source of load, crowding out
the requests that would have actually succeeded.

Worse, if many clients started calling the service at roughly the same
time (say, right after a deploy or a cache expiry), their retries stay
synchronized — they all back off, then all retry again at the same moment,
in lockstep. That's the thundering herd problem.

## Exponential backoff: better, but not sufficient alone

The standard first fix is exponential backoff: instead of retrying
immediately, wait progressively longer between attempts.

```
attempt 1: wait 100ms
attempt 2: wait 200ms
attempt 3: wait 400ms
attempt 4: wait 800ms
```

This reduces the total volume of retries hitting the struggling service
over time, which helps. But on its own, it doesn't fix the synchronization
problem — if every client backs off using the exact same schedule, they
still retry in lockstep, just at longer and longer intervals.

## Adding jitter

Jitter means adding randomness to the backoff interval so that clients
don't retry in sync with each other. A few common strategies:

- **Full jitter** — pick a random wait time between 0 and the calculated
  backoff value: `wait = random(0, base * 2^attempt)`. This spreads
  retries out the most, at the cost of some retries happening sooner than
  the "ideal" backoff would suggest.
- **Equal jitter** — keep half the backoff fixed and randomize the other
  half: `wait = base * 2^attempt / 2 + random(0, base * 2^attempt / 2)`.
  A middle ground between predictability and spread.
- **Decorrelated jitter** — each wait time is randomized based on the
  previous one rather than purely on the attempt count, which tends to
  produce a good spread without needing to track the attempt number
  precisely.

In practice, full jitter is a reasonable default unless you have a
specific reason to prefer tighter bounds on retry timing.

## Retries need a budget, not just a backoff curve

Backoff and jitter control the *timing* of retries, but you also need to
control the *volume*. A retry budget caps the fraction of a client's total
request volume that's allowed to be retries — commonly something like "no
more than 10% of requests may be retries, calculated over a rolling
window." Once you exceed the budget, further failures fail fast instead of
retrying. This protects the downstream service from being overwhelmed
precisely when it's already unhealthy, and it's the piece that pure
backoff-and-jitter doesn't address on its own.

## Idempotency is a prerequisite, not a detail

None of this is safe unless the operation being retried is idempotent —
retrying a non-idempotent write (like "charge this card" or "increment
this counter") can duplicate the effect of the original request if it
actually succeeded but the response was lost. The usual fix is an
idempotency key: the client generates a unique key per logical operation,
and the server deduplicates any request carrying a key it's already
processed. Without this, aggressive retrying trades one class of bug
(dropped requests) for another (duplicated side effects), which is
usually worse.

## The short version

- Immediate retries amplify load exactly when a system is least able to
  handle it.
- Exponential backoff spreads retries out over time but not across
  clients.
- Jitter breaks the synchronization between clients.
- A retry budget bounds how much of your traffic can be retries at all.
- None of it is safe without idempotency on the operation being retried.

Retries are a tool for handling transient failures, not a substitute for
fixing the underlying reliability of a dependency — and a naive
implementation can make the dependency's bad day measurably worse.
