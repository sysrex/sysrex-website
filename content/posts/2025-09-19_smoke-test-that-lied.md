+++
title = "The Smoke Test That Lied: Why \"It Ran\" Isn't the Same as \"It Passed\""

[taxonomies]
tags = ["Rust", "Distributed Systems"]
+++

I wrote a small standalone client to poke a websocket service I was building: connect, send some data, print whatever comes back. I ran it against a broken deployment to make sure it would catch the breakage. It printed an error message and exited 0. I stared at that for a solid minute — the tool had done exactly what I asked it to do, and that was the whole problem.

<!-- more -->

## A smoke test that can't fail isn't a smoke test

Here's roughly what the first version looked like:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let (ws, _) = connect_async(&url).await?;
    let (mut write, mut read) = ws.split();

    for chunk in &chunks {
        write.send(Message::Binary(chunk.clone())).await?;
    }
    write.send(Message::Text("done".into())).await?;

    if let Some(Ok(reply)) = read.next().await {
        println!("{reply:?}");
    }
    Ok(())
}
```

This is the shape almost every "quick script to check the deployment" starts as, and it's not wrong exactly — it does connect, it does send, it does print the server's reply. The bug is what it doesn't do: it never looks at what it printed. An error JSON body, a success JSON body, and garbage all take the same code path — `println!`, then a clean exit. The process's exit code, the one thing CI actually checks, has no relationship to what the server said.

I'd built a tool that could only ever tell me the connection worked. Whether the *operation* worked was left as an exercise for whoever happened to be reading the logs at the time, which in practice was nobody.

## The failure mode that's easy to miss

The obvious fix — check the reply, exit nonzero on an error status — covers the case where the server is honest about failing. It doesn't cover the case where the server can't tell you anything at all. Testing that case is what turned up something I hadn't expected:

A client can finish sending by either transmitting an explicit "done" message, or by just closing the connection. Both are reasonable ways to say "that's everything." But the server can only *reply* in the first case. The moment it receives a close frame from the peer, the underlying websocket library refuses to send anything further — not a policy choice on my end, a protocol-level guarantee that you don't write to a socket the other side has already told you it's done reading from. The work still happens: the buffered data still gets processed and stored. There is simply no wire left to send the confirmation back on.

So "the smoke test got silence" has two entirely different causes that look identical from the outside — the server is down or wedged, or the server did the work and structurally cannot tell you so — and a test that treats both as the same kind of failure is either too paranoid (flagging a client that intentionally closed early) or too lenient (treating a hung connection as fine because "no reply" is expected sometimes anyway). You have to pick, upfront, which path your test client is going to exercise, and only bail on silence for the path where silence is actually wrong.

## What the fixed version checks

The corrected client always sends the explicit "done" frame — the path where a reply is possible — and then treats anything other than a matching success reply as a failure:

```rust
let reply: Reply = tokio::time::timeout(timeout, read_reply(&mut read))
    .await
    .context("timed out waiting for reply")??;

match reply {
    Reply::Ok { bytes, chunks, key } => {
        anyhow::ensure!(bytes == expected_bytes, "byte count mismatch");
        anyhow::ensure!(chunks == expected_chunks, "chunk count mismatch");
        anyhow::ensure!(key.ends_with(&expected_hash), "content hash mismatch");
    }
    Reply::Error { message } => bail!("server reported an error: {message}"),
}
```

Four distinct ways to fail now produce four distinct nonzero exits with a reason on stderr: an explicit error reply, a reply that doesn't match what was actually sent (wrong byte count, wrong hash — the kind of bug that silently corrupts data while reporting success), a timeout, and a close with no reply at all. That last one, on the "done" path, is unambiguous: it means something is actually wrong, because I've already ruled out the one legitimate reason a reply might not show up.

## The general shape of the mistake

None of this is exotic — it's the same failure mode as a shell script piping through `grep` without `pipefail` and reporting success on a match count of zero. The pattern is: a verification tool that has to do real work to run (open a connection, send data, wait) accretes that complexity first, and the actual verification — did the thing I care about actually happen, correctly — gets added later, if at all, because by the time the plumbing works you've already seen it print something plausible-looking and moved on.

The uncomfortable part is that a smoke test like this is worse than not having one. No smoke test is an obvious, visible gap — someone eventually asks "wait, how do we know this works?" A smoke test that always exits 0 answers that question with false confidence, and false confidence doesn't get questioned nearly as often as an obvious gap does. It took a deliberately broken deployment for me to notice mine was doing that, which is a less reliable way to find out than I'd like.

## Sources

- [Post-Deploy Smoke Test failing on main](https://github.com/azmartone67/dchub-backend/issues/4982) — a real example of a smoke test that only started catching failures once `pipefail` was added.
- [Smoke Testing Strategies for Reliable Deployments](https://adhdecode.com/devops/advanced-topics-and-future-of-devops/smoke-testing-strategies/) — a smoke test should always pass on a healthy instance and always fail on an unhealthy one; anything less isn't earning its place in the pipeline.
