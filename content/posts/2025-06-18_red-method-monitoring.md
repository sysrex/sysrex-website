+++
title = "The RED Method: A Simple Starting Point for Monitoring Services"

[taxonomies]
tags = ["DevOps", "Monitoring"]
+++

Once you've got [Node Exporter](/posts/install-node-exporter-on-linux/)
reporting host-level metrics, the next question is usually: what should I
actually put on a dashboard for my services? The RED method is one of the
simplest, most reusable answers to that question.

<!-- more -->

## The three signals

RED stands for **Rate, Errors, Duration**. For any request-driven service
(an HTTP API, a gRPC service, a queue consumer), you track:

- **Rate** — how many requests is this service handling per second?
- **Errors** — how many of those requests are failing?
- **Duration** — how long are requests taking (usually as a distribution —
  p50, p95, p99 — not just an average)?

That's it. It sounds almost too simple, but the point is that these three
numbers, tracked per service and per endpoint, answer the vast majority of
"is this thing okay right now?" questions you'll ask during an incident,
without needing to know anything about the service's internals ahead of
time.

## Why it works

Averages hide problems. A service can have a perfectly reasonable average
latency of 80ms while 1% of requests take 8 seconds — and that 1% might be
exactly the requests that matter to whoever is complaining. Tracking
percentiles instead of averages, and errors as their own explicit signal
rather than something you infer from latency, means the dashboard tells
you what's actually happening rather than an average that no real request
experienced.

Rate matters because errors and latency mean different things at different
volumes. A 5% error rate on a service handling 10,000 requests per second
is a very different incident than the same 5% on a service handling 10.

## Instrumenting it with Prometheus

If you're already using Prometheus (which pairs naturally with Node
Exporter), the standard way to get all three RED signals from a single
metric is one histogram per endpoint:

```
http_request_duration_seconds_bucket{route="/api/orders", method="POST", status="200"}
```

From that single histogram, you can derive all three signals with PromQL:

```promql
# Rate — requests per second over the last 5 minutes
sum(rate(http_request_duration_seconds_count{route="/api/orders"}[5m]))

# Errors — percentage of requests returning 5xx
sum(rate(http_request_duration_seconds_count{route="/api/orders", status=~"5.."}[5m]))
  /
sum(rate(http_request_duration_seconds_count{route="/api/orders"}[5m]))

# Duration — p95 latency
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{route="/api/orders"}[5m])) by (le)
)
```

Most HTTP frameworks have a Prometheus middleware that exposes exactly this
kind of histogram with almost no setup — the labels (`route`, `method`,
`status`) are the part worth getting right, since they're what let you
slice rate/errors/duration per endpoint instead of just per service.

## Where RED stops being enough

RED is a starting point, not the whole picture — it's built for
request-driven services, so it doesn't map cleanly onto batch jobs, queue
depth, or resource saturation (CPU, memory, disk). For that, RED is usually
paired with the **USE method** (Utilization, Saturation, Errors), which is
aimed at resources rather than services — closer to what Node Exporter
itself reports on. Between the two, you get a reasonable default answer to
both "is my service healthy" and "is my infrastructure healthy," without
having to invent bespoke metrics for every new system you deploy.
