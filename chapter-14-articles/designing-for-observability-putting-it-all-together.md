# Designing for Observability: Putting It All Together

## Learning Objectives

- Wire logs, metrics, traces, health, and flags into one story: the correlation id is the glue that joins a metric spike to the trace and the log lines.
- Order the layers of instrumentation so a production question ("why is checkout slow for region X") is answerable in minutes, not a sprint.
- Decide what must be instrumented at day one versus what can wait, and make the maturity a checklist with a real owner.

## Introduction

Each of the last eight articles gave you a tool: structured logs, audit records, metrics, health probes, retries, breakers, rate limits, flags. Observability is what the tools are for when they are wired together. It is the property that the system answers production questions: what is happening now in aggregate (metrics), what exactly happened for a request (logs and traces), and whether an instance is worth giving work (health). Metrics tell you checkout is slow; a trace tells you the slow page was a payment call; the log tells you the payment call got a 500 from a specific gateway; the audit says the actor and the moment. Without the wiring, each of those is a separate graph and the incident is a scavenger hunt across five UIs.

## Problem Statement

An on-call engineer gets a page: checkout p99 latency is red. The dashboard shows the spike but not the cause. To find the cause they open a second tool for traces, a third for logs, a fourth for the deployment history, and they know only that the spike started four minutes ago. The reason it takes forty minutes instead of four is that the correlation id never bridged the gap: the latency metric is not joined to the trace id of a slow checkout, so a dashboard click cannot jump from "spike" to "the exact request" to "the exact log line." The parts were all built. The wiring that made them one system: that part was never designed, and the missing design, not any missing tool, is the reason an on-call ends in a scramble.

## Core Concept

### The three pillars and one joiner

The metaphor that holds: metrics are the "is something wrong," logs are "what happened," traces are "where did the time go," health is "who can serve," and the correlation id is the single field that lets them join. A request enters carrying request id. That id is in the log every line via the MDC, it is the trace id that labels every span, it appears on the metrics when they are labeled by the caller. The whole observability architecture is: instrument once with the id, and make every tool keyed by the same id, so any screen can pivot to any other through one field.

The wiring, as a picture:

Diagram: one request id flows through the service layers and keys every observability signal for the same call.

<svg width="900" height="380" viewBox="0 0 900 380" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="ob" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#333"/>
    </marker>
    <marker id="obg" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#2f6f3e"/>
    </marker>
  </defs>

  <rect x="40" y="160" width="170" height="70" rx="6" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="125" y="196" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">checkout</text>
  <text x="125" y="216" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">request id enters</text>

  <rect x="380" y="160" width="170" height="70" rx="6" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="465" y="196" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">service chain</text>
  <text x="465" y="216" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">id across spans</text>

  <rect x="720" y="160" width="150" height="70" rx="6" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="795" y="196" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">gateway</text>
  <text x="795" y="216" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">id preserved</text>

  <path d="M 210 195 L 372 195" stroke="#333" stroke-width="2" fill="none" marker-end="url(#ob)"/>
  <path d="M 550 195 L 712 195" stroke="#333" stroke-width="2" fill="none" marker-end="url(#ob)"/>

  <rect x="40" y="300" width="170" height="60" rx="6" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="125" y="334" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">logs keyed</text>

  <rect x="230" y="300" width="170" height="60" rx="6" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="315" y="334" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">metrics tagged</text>

  <rect x="420" y="300" width="170" height="60" rx="6" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="505" y="334" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">trace spans</text>

  <rect x="610" y="300" width="150" height="60" rx="6" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="685" y="334" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">audit events</text>

  <path d="M 125 228 L 125 292" stroke="#8a6d00" stroke-width="1.5" fill="none" marker-end="url(#ob)"/>
  <path d="M 315 228 L 315 292" stroke="#2f6f3e" stroke-width="1.5" fill="none" marker-end="url(#obg)"/>
  <path d="M 465 228 L 505 292" stroke="#8a6d00" stroke-width="1.5" fill="none" marker-end="url(#ob)"/>
  <path d="M 685 228 L 685 292" stroke="#333" stroke-width="1.5" fill="none" marker-end="url(#ob)"/>

  <text x="600" y="285" text-anchor="middle" font-family="Arial" font-size="11" fill="#555" transform="rotate(0)">same request id on every signal</text>
</svg>

The single field is what makes the jump possible. A dashboard 's `checkout 500s` panel is grouped by `requestId`; clicking on a request id opens the trace; the trace shows the failing span; the span includes the log line from the MDC, and that log line has the stack. The query path is a chain of joins on one id, and it is built at the time you wrote the filter that puts the id in the MDC. That filter is the whole architecture in miniature.

### Layered to answer a question

Each layer answers a different question at a different cost:

- Metric: "is X broadly ok right now." Cheap, aggregate, always on. It tells you a problem started, not which request.
- Trace: "why was this request slow." Expensive because it instruments every call, sample it.
- Log: "what exactly happened for this request." Only the request that failed or logged.
- Health: "is this instance able to serve now." The gate the router asks.
- Audit: "who was the actor when this happened": a join that only the dispute needs.

The anti pattern is using a log to answer "is it broadly OK" (a log line has no aggregate, no percentile) and using a metric to answer "what exactly happened" (a metric has no stack). Knowing which pillar answers which question is the skill: "is checkout slow" is a metric, "why is this checkout slow" is a trace first and a log second, and "who did this" is an audit. The mismatch, answering a trace question with a metric, is the mark of a postmortem that stalls.

### Sampling is a design, not a default

Traces are the expensive pillar; the answer to "instrument everything with a trace" is a budget you pay in throughput. The standard is sampling: keep a full trace of, say, 1 in 100 requests, and keep every trace that errored or was slow (the tail matters most for debugging). The reason errors are kept at full is that error traces are the ones you will need when the incident page fires. The failure mode: sample only by percentage, which keeps 1 of every 100 including all the fast happy paths, and drops exactly the slow pathological one on the day you need it. The right filter is "keep all slow + all erroring, keep 1% of the rest."

### The runbook and the dashboard as code

The pieces are only as good as the wiring, and the wiring is code. The mature team stores the dashboards, the alert rules, and the runbooks in the repo, reviewed like anything else, and the alert rule references the runbook in its annotation. The discipline: an alert has an owner and points to a document, and the dashboard has an owner and a warning history. The tell that a team has observability as an actual system is not the tool shape, it is that the alert rules have an owner that can reason about the budget.

An SLO with an error budget from the metrics chapter is the load that makes alert rules, the breaker policy, and the health probe cooperate: when error budget burns, the alert fires, the on-call opens the trace view from the alert, and the breakers and retries stop as configured in the earlier chapters. The observability is what all those policies render as a decision, an open alert.

### Maturity is a checklist that someone owns

The first stage is firefighting: logs exist but nobody looks, health is the default, no SLO. Maturity steps through a tiered shape:

- Tier 1: logs and basic health, alerts on hard 500s.
- Tier 2: metrics with golden signals, traces sampled, the correlation id joins everything, runbooks written.
- Tier 3: SLOs with error budgets, alert rules meaning something, dashboards with named owners, releases behind flags.

Most teams plateau at tier 2, because the hard twenty percent is not the tool, it is the wiring: the id everywhere, the runbook written, the owner named. Observability is measured by two questions: who owns the dashboards, and can the correlation id reach every pillar. Answer those two and the tools stop being decorative wallpaper.

## Real Production Usage

The concrete stack is the one most teams run: OpenTelemetry instrumenting the JVM and producing the trace id, and the same id landing in the log MDC to serve as the request id. Logs go to Loki or ELK, metrics to Prometheus, dashboards to Grafana, and the three join on the id. In Spring Boot, `spring-boot-starter-actuator` plus Micrometer exposes metrics and `spring-boot-starter-otel` carries the trace. The outcome is a single surface: "this alert fired, here is the budget burn, here is the dashboard, here is the trace of a slow page, here is the log line with the stack," reachable without changing tools.

## Common Mistakes

1. **Instrumentation of volume, not structure.** Every service logs freely, but the request id is not propagated across them, so a metric spike has no trace and a trace has no log. The wiring of the id is the architecture; without it, the volume is storage.
2. **Sampling that strips the tail.** Sampling only by rate means the slowest and the broken traces are exactly the ones dropped. The tail matters most for debugging, so keep every slow and erroring trace and sample only the healthy majority.
3. **Tool proliferation ahead of wiring.** Adding a second tracing vendor or a new metering backend before the existing tools can join on the id is not observability, it is spending. The correlation id is the product; the dashboards are the byproduct.

## Interview Perspective

An interviewer who asks "how do you design observability for a service" wants to hear the layers wired together, not the vendor list. The strong answer: "a metric for the big picture, a trace for the slow request, a log for the exact error, health for the router, audit for the story, all joined by one correlation id so any signal can open any other, sampling that keeps the slow and erroring whole, and alerts that point at a runbook." The weak answer lists four products and calls it done. Resolve their favorite follow-up is "you sampled traces and dropped the failing ones, how do you debug it?" The answer is the filter: keep the offenders whole.

## Knowledge Check

1. Checkout p99 spikes, and the trace sampling dropped the failing four hundred requests. As the on-call, name the two ways you still answer "which request, and why," and the fix so you never face this again.
2. Every log carries the request id, but the checks list spike with no drill-down. Name the minimum path to go from the metric to a trace, and state where it breaks if the id never crosses the service boundary.
3. A runbook lives in a draft on somebody's laptop. Why is that the same sentence as "observability is not code," and what does owning it as code buy you?

## Key Takeaways

- Observability is the join: the same correlation id in logs, metrics, traces, and health is what turns four graphs into one incident response.
- The plumbing and sampling are design decisions made with owners, and the tail must be kept/collected.
- Dashboards, alerts, and runbooks are code, owned by named people, tied to an SLO, or they are shelf objects.

## What's Next

This chapter gave you the reliability tools that make a live system answerable, logging, audit, metrics, health, retries, breakers, rate limits, flags. The next chapter turns the lens completely: LLD Case Studies, which works through the classic systems, and instead of steadying a running service you will design a system from the ground, the entities, the classes, the patterns, and the state machine that a whiteboard case actually wants. The reliability ideas you built here, the structured log, the idempotent retry, the health probe, they reappear inside every case, but the questions become structural first, correctness and shape, with observability as the layer you design in, not the retrofit.

---

This article explains observability as the wiring that joins logs, metrics, traces, and health into one incident record. It argues that a metric with no clickable trace and log is not instrumented, and the missing layer is correlation, not another tool.
