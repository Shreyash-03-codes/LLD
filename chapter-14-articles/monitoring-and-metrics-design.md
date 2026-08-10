# Monitoring and Metrics Design

## Learning Objectives

- Pick the right metric type for a question: a counter for counts, a gauge for instantaneous values, a histogram for distributions.
- Name the four golden signals and map each one to a concrete measurement with a concrete alert.
- Turn a metric into an SLO and an error budget, and know why an alert with no owner is a liability.

## Introduction

Monitoring is the practice of deciding, in advance, what will tell you the system is broken, and wiring that answer into a graph and a pager. Metrics are the language of monitoring: a number with a name and labels, sampled over time. The discipline exists because the alternative, watching logs for trouble, does not scale past a handful of instances. A metric is cheap to collect, cheap to store, and cheap to aggregate, which makes it the right tool for the question "is this thing broadly OK right now?" Logs answer "what exactly happened," metrics answer "is anything wrong," and never the two shall change places.

## Problem Statement

The service went down at 2 a.m. and nobody knew until customers called. The dashboard exists, but it shows a wall of graphs that all look the same, a green line that never moved, and one graph whose axis is in bytes but nothing says bytes of what. The alerting rules, if they exist, are thresholds someone guessed once and never revisited. By the time an engineer opens the dashboard, the incident is over and the story has to be reconstructed from the logs and the memory of whoever was awake. The failure is not that the system had no observability; it is that nobody decided what "broken" looks like before it broke.

## Core Concept

### Metric types are a vocabulary

Three metric types answer most questions, and choosing wrong produces a graph that lies.

- **Counter**: a monotonically increasing number, requests served, errors, bytes sent. You sample it and compute rate of change. Counters only go up, and they reset on restart, so the graph is a rate, not a level.
- **Gauge**: a value that goes up and down, current queue depth, active connections, memory in use. You read it as-is; it represents an instantaneous state.
- **Histogram**: a sample of observations, request latency, payload size, with buckets. A histogram gives you percentiles, which is how you answer "what is the p99 of checkout," which no average can.

The common error is using an average for latency. An average of request latency hides the tail: if 95 requests take 10 ms and 5 take 5 seconds, the average is 259 ms, which looks healthy and describes almost nobody. Percentiles are the vocabulary of latency, and p50 tells you the median user, p99 the tail, p99.9 the users who churn.

```java
// micrometer naming: a counter for errors, a histogram for latency
Counter.builder("http.server.requests.errors")
    .tag("method", "POST").tag("path", "/checkout").register(meterRegistry);

Timer.builder("http.server.requests.duration")
    .publishPercentiles(0.5, 0.95, 0.99)
    .register(meterRegistry);
```

### The four golden signals

Google's four golden signals remain the best checklist for what to watch on a service:

- **Latency**: how long to serve a request. Measure percentiles, and distinguish success from failure latency; a failing fast endpoint is not the same as a slow healthy one.
- **Traffic**: how much demand the system is getting. Requests per second, concurrent users, bytes in and out. Traffic tells you whether a change in latency is a load change or a bug.
- **Errors**: the rate of requests that fail, with an explicit definition of failure, HTTP 5xx, business error codes, timeouts. Error rate is the signal most tied to user harm.
- **Saturation**: how full the resource is, queue depth, CPU, connection pool usage. Saturation tells you how close you are to the cliff before latency climbs.

The signals interact. An alert on errors alone is late if saturation climbs first; an alert on latency alone is confusing when traffic spikes. The practical setup is to watch traffic to interpret everything else, alert on error rate and latency, and treat saturation as the leading indicator.

### Cardinality is the hidden tax

Every label on a metric multiplies the number of distinct series. `http.server.requests` by method and path is fine. The same metric by user id or by cart item id is a cardinality explosion that blows up the metric store and the dashboard render time. The rule: labels should be finite and bounded, method, path, region, status class, never an id that grows with usage. Teams discover cardinality when dashboards start timing out or the metrics backend starts dropping series. A metric with a user id is a log wearing a costume.

### SLOs and error budgets

An SLO is a target written as a number over a window: "checkout p99 latency under 800 ms over 30 days." The error budget is the allowed failure in that window, and the point of the budget is that it is spendable. When the budget is nearly exhausted, you stop risky releases; when it is healthy, you can move. The SLO turns monitoring from a report into a decision. Without an SLO, an alert is a threshold with no agreed meaning, and "how much downtime is acceptable" is a question nobody decided. The alerting rule derives from the SLO: alert when you are burning budget faster than planned, not when a single request crosses a threshold.

```java
// alerting on error budget burn: 5% of requests failing over 6h
boolean burning = (errorRate * windowMinutes) > (budget * windowMinutes);
```

This is the shape of a real rule: it measures the trend against the agreed budget, not a single spike.

## Real Production Usage

Micrometer is the standard Java metrics facade; it exposes counters, gauges, histograms, and timers and ships them to Prometheus, Datadog, Grafana, and others. Spring Boot auto-configures Micrometer and exposes `/actuator/prometheus`, so a service ends up with JVM memory, thread pools, HTTP traffic, and DB pool usage for free, and your own counters layer on top. Alerting runs in Prometheus Alertmanager or the hosted equivalent, and dashboards in Grafana. The mature stack treats the golden signals as the spine of the dashboard and the SLO as the source of the alert rules.

## Common Mistakes

1. **Averaging latency.** The average hides the tail. Percentiles or nothing, and p99 without p50 hides the median.
2. **Cardinality creep.** Adding a user id label to a metric is a small change in the code and a large change in the store. Keep labels bounded.
3. **Alerts with no owner and no SLO.** A threshold someone set once and forgot either fires constantly, so nobody believes it, or never fires, so it was theater. An alert without an owner and a meaning is a liability.

## Interview Perspective

The question is "how do you monitor this service?" The weak answer lists tools. The strong answer names the four golden signals, picks the metric types, and derives the alert from an SLO: "I watch latency by percentile and error rate, traffic to interpret both, and saturation as the leading edge, and I alert when error budget burns, not on a single spike." Interviewers follow up with "what do you do about the p99" and "what is a counter vs a gauge" to check you understand the types and the tail, not just the dashboard names.

## Knowledge Check

1. An endpoint's average latency is 200 ms and the p99 is 4 seconds. What does the product feel like, and why did the average hide it?
2. You add a `userId` label to a request metric. What breaks, and when does it break, hours later or months later?
3. The error budget for the month is nearly gone. What does the SLO say you should do, and why is that the point of having a budget at all?

## Key Takeaways

- Counters, gauges, and histograms answer different questions; latency is a distribution and belongs in percentiles.
- The four golden signals, latency, traffic, errors, saturation, are the checklist; saturation is the leading indicator.
- An SLO with an error budget is what turns a metric into a decision, and an alert with no owner is a liability.

## What's Next

Metrics tell you something is broadly wrong, but they are sampled and aggregate; they cannot tell you whether this specific instance can take traffic. The next article covers health checks, which is how a load balancer and an orchestrator ask a single instance "are you alive, and are you ready for work?" before they route anything to it.

---

This article explains monitoring as deciding in advance what broken looks like, using counters, gauges, and percentiles. It argues that latency is percentiles, cardinality the hidden tax, and an SLO with an error budget turns an alert into a decision.
