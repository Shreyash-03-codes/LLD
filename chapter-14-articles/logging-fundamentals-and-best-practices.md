# Logging Fundamentals and Best Practices

## Learning Objectives

- Produce a log line that works as a self-contained machine-readable record: timestamp, level, class, message template, correlation fields, values as fields.
- Use SLF4J placeholders and the MDC so structure and correlation come from the library, not from string concatenation scattered through the code.
- Treat log levels as a contract between the code and the person paged, not as a gauge of how the developer felt.

## Introduction

Logging is the oldest observability and the one most teams treat as free. It is the output your pager reads, your dashboards alert on, and your debugging session starts from. The framing that hurts most is "logs are a text file I can grep." That framing lets every decision slide, because a text file has no schema. The working model is the opposite: a log stream is a queryable set of structured records, and the schema is decided at the call site, line by line, by whoever writes the log.

## Problem Statement

Midnight. The pager fires because checkout fails intermittently. You open the logs to find the failed order. What you find is a wall of prose: one message embeds the order id inline, another does not, a stack trace was truncated because a catch handler concatenated `e.toString()`, timestamps came from two instances in different timezones, and none of the lines carry the user id. Reassembling one failed order takes minutes of searching. A structured record with the order id as a field would have answered in one query. The failure is not that the log exists. It is that logging had no schema or key, so the record of the incident is a scatter of sentences nobody can group or join.

## Core Concept

A healthy log line is a record with stable parts: a timestamp, a severity, the logger name, correlation fields, a message template, and key-value context. This shape is not decoration. The message is the noun, the values are its attributes.

```java
private static final Logger LOG = LoggerFactory.getLogger(OrderService.class);

LOG.info("order created orderId={} customerId={}", orderId, customerId);
```

The logger name says the class. The placeholders fill a template without string concatenation. The values stay as named fields on the line, and when the stream becomes JSON they become queryable columns. Get the shape right and everything downstream, search, grouping, alerting, is cheap. Get it wrong and every downstream tool is the same regex-over-prose hack, rebuilt each time.

### The template is the schema

Two developers write the same log. One writes `"order created order=" + orderId + " for customer=" + customerId`. The other writes the template form. Two years later, whoever sums "orders per minute" searches for `order created` and gets clean grouping from the second codebase and a mess of variants from the first. The template is the schema of your log stream, and it is set by writing messages as atoms, not sentences. A hundred one-off wordings of the same fact is how log grouping silently dies.

### Levels are a contract with the pager

The levels are TRACE, DEBUG, INFO, WARN, ERROR. Plenty of teams treat them as "how bad I felt," and that is exactly what beats them. A level exists so a reader or a dashboard can decide what to do without parsing meaning out of prose.

- TRACE and DEBUG: off in production by default. For the developer who turned them on in the moment.
- INFO: a significant transition. Order created, payment captured. Not "request number 42 arrived." INFO is a milestone, not a heartbeat.
- WARN: the request survived but degraded. A handled failure, a retry, a slow path, nearing quota. A WARN stream that is never empty and never actionable is noise.
- ERROR: the request failed and nothing recovered it. An ERROR must be something a person can act on or acknowledge.

The contract that sticks: an ERROR is a fact a human may need to look at. Expected, handled, routine conditions are not ERROR; they are WARN at best. The failure mode that does the most damage is an error rate that spikes because expected behavior logs at ERROR, while the real internal failure logs at INFO for no one. Then the page gets filed as expected, and the actual defect never surfaces. This is how a system breaks silently: the level is wrong, so nobody is looking at the line that matters.

The other trap is DEBUG. Placeholders are cheap because logback skips work when the level is off, but the arguments are still evaluated before the call. When argument construction is genuinely expensive:

```java
if (LOG.isDebugEnabled()) {
    LOG.debug("plan for city {} detail {} rows", city, planner.expensiveRender(500));
}
```

Hoist the expensive build under the guard. Do not guard everything, the guard itself is noise. Guard only the rare and costly cases, and name the cost when you review the diff.

### JSON over prose

Production logs are consumed by machines. A JSON object on one line names every field, so a dashboard can sum the `status` field or group by `region` without a regex. A prose line is payload, and each new report costs a new regex and a new parser. That is the real argument for JSON: it collapses the cost of the next question you will ask. The Java side stays unchanged; the encoder writes JSON, the code fills structured fields.

### The MDC for correlation

A user's request flows through a filter, a controller, several services, maybe threads, maybe a call to another service. To reassemble that one request out of a shared stream, every line must carry a piece of identity. That is the job of the MDC (Mapped Diagnostic Context), a per-thread map that the encoder flushes into every line logged on that thread.

```java
public class CorrelationFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String requestId = extractOrCreate(((HttpServletRequest) request).getHeader("X-Request-Id"));
        MDC.put("requestId", requestId);
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

Capture at the edge, clear in a finally. The finally is the whole discipline. Application servers and async appenders reuse threads, so an uncleared MDC bleeds request A's fields into request B's logs. The symptom is a requestId that belongs to someone else's order, or a trace that suddenly follows a random user, and it disappears on a redeploy, which makes it look like a phantom. `MDC.clear()` in the `finally` is the invariant, not a nicety.

### Exceptions with their stack, always

When you log a caught exception, there are two forms and they are not close.

```java
LOG.error("payment failed orderId={}", orderId, e);            // keeps the stack
LOG.error("payment failed orderId={} errorText={}", orderId, e.getMessage());   // a summary, not a record
```

The second is what teams write when they want a clean line: the exception flattened to text. When the failure recurs and the line only says `TimeoutException: read timed out`, there is no frame, no class, no caller, just a sentence. That is a dead end for the human who paged in. The fix is trivial, log the throwable last. The rule about volume: log the exception once, at the boundary where the decision was made. Five layers of catch-and-log print five copies of the same stack and bury the one that matters. Boundary logging plus propagation is the clean shape.

### Async logging: keep the I/O off the request thread

A line that writes to disk synchronously in the request path adds latency. The norm is an async appender: events go to a bounded queue and hit the disk on a writer thread. The abuse is the queue. When it overflows at a burst, dropped default behavior throws away newer events, which is the fine moment to lose your error. There is a deliberate choice to block the sender instead of dropping. Blocking costs latency, dropping costs the record of the incident. Most teams take the default that drops, and most do not repeat. The logging article is the moment to say it: choose what loses the error at the highest load, and make that explicit.

Async logging also reorders output across threads, so never treat the stream as ordered. The "last line = latest event" assumption breaks exactly when threads are busy. The timestamp plus correlation fields are your durable order, and the trace id is what glues a story together.

### Privacy is a logging property

Two families of PII leak constantly: request logs that dumped the body, and session tokens or full headers logged by a debug helper that a pull request forgot. The fix is at the framework edge: never log the body by default, keep tokens out of the line, and treat anything you emit as inspectable. The person reviewing a log line in a production debugging console is not necessarily entitled to the customer's session token just because it is there. log the identifier, not the token.

## Real Production Usage

The production stack is stable: SLF4J as API, logback or Log4j2 as implementation, logstash-logback-encoder for the JSON output, and a filter that fills the MDC with a request id and a trace id, plus a collector such as Loki, Graylog, or the ELK family that decodes the JSON and indexes the fields. That trace id is what later joins a log line to its trace, and the request id is what joins a log line back to a dashboard panel. In teams that have observability, a structured log with a status field is the raw data of a graph: the stack of `http.server.requests` lines per minute is what the rate panel is built from, and a buried 500 is a countable event, not lost prose.

## Common Mistakes

1. **Manual string factorization of a throwable.** Calling `e.getMessage()` and concatenating it into the log text removes the stack. If the debug setup fails, the message line is all that survives and the incident is unconnected.
2. **Logging inside a hot loop per item.** A game loop or an `items.forEach(item -> LOG.info(...))` produces volume by design. Log the aggregate and report the outcome once.
3. **An async queue with a drop policy you never read.** When the peak of the failure coincides with the buffer full, the default drop policy starts to lose the error at the very part of the timeline that matters.

## Interview Perspective

The question is "how do you design logging?" The weak answer is "log everything." The strong answer: "structured log with levels set by the consumer, correlation via MDC at the edge, a clear in finally, an exception that keeps the stack, JSON via the encoder, a bounded but deliberate async appender, and never logging the body." Interviewers follow up with "what happens when your queue overflows" to check that you know the queue is finite and have chosen recovery, and "how does a log connect to a trace" to see whether you know the request id joins the stream.

## Knowledge Check

1. Two services exchange a call, but the request id is generated and dropped at the boundary. What breaks in the log stream, and where did the fix belong?
2. A page fires because the error logs spike at ERROR, but the real failure was handled and logged at INFO. Describe how that is even possible and what the contract fix is.
3. A DevOps asks you to speed up the request path and you find a synchronous append per log. Sketch what the queue behavior options are when you make it, and say which one you turn to at the cost of losing an error line.

## Key Takeaways

- Log lines are records, level plus template plus fields, or they are noise paid for by a debug effort.
- Levels are for the person paged, not for the sender. Anything that will make a query follow a field must be logged with that exact field.
- The MDC correlation and an exception as a throwable are what make your past debugging a story, and the queue restoration and choice of queue are what keep the error's moments on record.

## What's Next

## What's Next

This article covered the mechanics of a troubleshooting log: level discipline, structure, correlation, exceptions, routing. The next article shifts who reads the line and why it exists: audit logging is the append-only record of who did what and when, built to survive restarts and tampering, which makes it a different job from debugging and a different set of engineering decisions.

---

This article explains the production log line as a structured record: template, fields, MDC, and a preserved stack. It argues failures come from deciding level and schema ad hoc at the call site, and the line is a contract with the pager.
