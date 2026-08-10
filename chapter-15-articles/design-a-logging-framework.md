# Design a Logging Framework

## Learning Objectives

- Design a small library, not a small application, and learn how the client contract (the `Logger` API) stays stable while the internals vary freely.
- Understand the producer-consumer core: application threads produce log records, one writer thread consumes them, and the queue between them is the whole concurrency story.
- Place the two classic extension seams, appenders and formatters, so that adding a sink or changing the output shape never touches the logging call sites.

## Introduction

Every other case study in this chapter models a thing the users interact with. A logging framework models the code your own users interact with, which changes the design completely. The client of a library is an engineer calling `logger.info("order " + id + " shipped")` from a hot path, and that engineer does not care about your architecture. They care that the call is cheap, that it does not throw, that it never blocks their request, and that the log line actually appears somewhere. The interviewer's job is to see whether you can design a system whose extension points are open to change without ever breaking that one-line contract. This is also the first case study with a genuinely mandatory concurrency story: applications log from many threads at once, and losing a log line is a silent, unacceptable failure.

## Requirements Gathering

Functional requirements:

- The framework exposes a logger with leveled methods: debug, info, warn, error.
- Each log statement carries a timestamp, level, message, and the source class or thread.
- Logs can be written to multiple sinks: console, file, and optionally a remote collector.
- Each sink can be configured with a minimum level, so a file may capture debug while the console only shows warn and above.
- A log line can be formatted differently per sink.

Non-functional requirements:

- Logging from a hot path must be cheap and must never block the calling thread when the sink is slow.
- No log record may be lost because two threads wrote concurrently.
- The framework must degrade gracefully: a broken sink must not take down the application.

Assumptions to state out loud: configuration is done once at startup, so the framework does not need hot-reload of levels; the remote collector sink exists in the interface but its transport is out of scope; there is no structured logging or JSON schema negotiation; and synchronous logging is acceptable when a caller explicitly opts into it. Cut hot-reload and cut the wire protocol. If you do not, you will spend the interview on config parsing instead of the queue.

## Identifying Core Entities

The entity list is the skeleton of every real logging library, which is a good sign that the design is on the right track.

| Entity | One-line responsibility |
| --- | --- |
| `LogLevel` | The ordered severity enum that drives filtering. |
| `Logger` | The client-facing API; the only class application code sees. |
| `LogRecord` | The immutable snapshot of one log statement. |
| `Appender` | A sink that writes records somewhere; console and file implementations. |
| `Formatter` | Turns a record into the text a sink emits. |
| `LoggerConfig` | Holds the level, the appenders, and their formatters. |
| `LogDispatcher` | The background thread and its queue that decouple producers from sinks. |

The two seams, `Appender` and `Formatter`, are the entire extension story. The dispatcher is the entire correctness story. Everything else is glue.

## Class Design

Start at the client boundary. `LogLevel` is an enum whose ordering matters, because the whole filter logic is a comparison.

```java
public enum LogLevel {
    DEBUG, INFO, WARN, ERROR
}
```

`LogRecord` is an immutable snapshot. It exists so that the record captured at the call site, with its timestamp and thread name, is exactly the record that gets written later, even if the sink is slow and the world has moved on.

```java
public class LogRecord {
    private final Instant timestamp;
    private final LogLevel level;
    private final String message;
    private final String source;
    private final String threadName;

    public LogRecord(LogLevel level, String message, String source) {
        this.timestamp = Instant.now();
        this.level = level;
        this.message = message;
        this.source = source;
        this.threadName = Thread.currentThread().getName();
    }

    public Instant getTimestamp() { return timestamp; }
    public LogLevel getLevel() { return level; }
    public String getMessage() { return message; }
    public String getSource() { return source; }
    public String getThreadName() { return threadName; }
}
```

`Appender` is the sink interface, and `Formatter` is the shape of its output. The split is that an appender knows where to write and a formatter knows what the bytes look like, so you can rotate a file appender without touching the formatter and restyle the output without touching the appender.

```java
public interface Appender {
    void append(String formattedLine);
    void close();
}

public interface Formatter {
    String format(LogRecord record);
}

public class ConsoleAppender implements Appender {
    public void append(String formattedLine) {
        System.out.println(formattedLine);
    }
    public void close() { /* nothing to flush for stdout */ }
}

public class FileAppender implements Appender {
    private final BufferedWriter writer;

    public FileAppender(String path) {
        try {
            writer = Files.newBufferedWriter(Path.of(path), StandardOpenOption.APPEND);
        } catch (IOException e) {
            throw new IllegalStateException("Cannot open log file", e);
        }
    }

    public void append(String formattedLine) {
        try {
            writer.write(formattedLine);
            writer.newLine();
        } catch (IOException e) {
            // never let logging take down the application
            e.printStackTrace();
        }
    }
    public void close() {
        try { writer.close(); } catch (IOException ignored) { }
    }
}

public class PatternFormatter implements Formatter {
    public String format(LogRecord record) {
        return String.format("%s %s [%s] %s - %s",
            record.getTimestamp(), record.getLevel(),
            record.getThreadName(), record.getSource(), record.getMessage());
    }
}
```

`LoggerConfig` groups the level and the sinks for one logger name. The per-sink minimum level lives here, which is what makes "console only shows warn, file captures debug" possible: filtering happens at the boundary, before the record crosses to a sink.

```java
public class LoggerConfig {
    private final LogLevel level;
    private final Map<Appender, LogLevel> appenders = new LinkedHashMap<>();

    public LoggerConfig(LogLevel level) { this.level = level; }

    public void addAppender(Appender appender, LogLevel minLevel) {
        appenders.put(appender, minLevel);
    }

    public boolean enabled(LogLevel recordLevel) {
        return recordLevel.ordinal() >= level.ordinal();
    }

    public boolean acceptedByAppender(Appender a, LogLevel recordLevel) {
        return recordLevel.ordinal() >= appenders.get(a).ordinal();
    }

    public Map<Appender, LogLevel> getAppenders() { return appenders; }
}
```

Now the core of the case study: the dispatcher. The producers are the application threads calling `logger.info(...)`. The consumer is one background thread. Between them sits a `BlockingQueue`. The producer enqueues and returns immediately; the consumer drains and writes. This is the producer-consumer pattern, and it is the answer to the "logging must never block my request" requirement.

```java
public class LogDispatcher {
    private final BlockingQueue<LogRecord> queue = new LinkedBlockingQueue<>(1024);
    private final ExecutorService writer = Executors.newSingleThreadExecutor(r ->
        new Thread(r, "log-writer"));

    public void dispatch(LogRecord record) {
        queue.offer(record); // non-blocking; drop if full rather than stall a request
    }

    public void start() {
        writer.submit(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    LogRecord record = queue.take();
                    write(record);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
    }

    private void write(LogRecord record) {
        LoggerConfig config = LoggerRegistry.get(record.getSource());
        if (config == null) return;
        for (Map.Entry<Appender, LogLevel> e : config.getAppenders().entrySet()) {
            if (config.acceptedByAppender(e.getKey(), record.getLevel())) {
                e.getKey().append(new PatternFormatter().format(record));
            }
        }
    }
}
```

The `queue.offer` with a bounded queue is a deliberate choice, and it is the kind of choice interviewers ask about. A bounded queue means the framework will drop records under overload rather than stall application threads, which is the right priority for a logging library. If the interviewer pushes for no drops, the answer is a larger bounded queue and a policy decision about what matters more, which is exactly the trade-off discussion the case study exists to provoke.

`Logger` is the thin client. It enforces the level check, builds the record, and hands it to the dispatcher. It never touches appenders or formatters. That is the contract application code depends on.

```java
public class Logger {
    private final String name;

    private Logger(String name) { this.name = name; }

    public static Logger get(String name) { return new Logger(name); }

    public void info(String message) {
        log(LogLevel.INFO, message);
    }
    public void error(String message) {
        log(LogLevel.ERROR, message);
    }

    private void log(LogLevel level, String message) {
        LoggerConfig config = LoggerRegistry.get(name);
        if (config == null || !config.enabled(level)) {
            return;
        }
        DispatcherHolder.dispatch(new LogRecord(level, message, name));
    }
}
```

Diagram: the producer-consumer pipeline. Call sites enqueue an immutable record and return; one writer thread drains the queue and fans out to per-sink filters and formatters.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 930 430" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah2" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="930" height="430" fill="#ffffff"/>

  <text x="450" y="30" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">Producer-consumer logging pipeline</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah2)">
    <line x1="170" y1="96" x2="216" y2="135"/>
    <line x1="170" y1="156" x2="216" y2="160"/>
    <line x1="170" y1="216" x2="216" y2="185"/>
    <line x1="400" y1="165" x2="446" y2="165"/>
    <line x1="620" y1="225" x2="686" y2="188"/>
    <line x1="765" y1="196" x2="765" y2="281"/>
    <line x1="765" y1="196" x2="765" y2="356"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="20" y="70" width="150" height="52" rx="6" fill="#f1f5f9" stroke="#cbd5e1"/>
    <text x="95" y="91" text-anchor="middle" font-weight="bold" fill="#334155">Thread-A</text>
    <text x="95" y="110" text-anchor="middle" font-size="12" fill="#64748b">logger.info(...)</text>
    <rect x="20" y="130" width="150" height="52" rx="6" fill="#f1f5f9" stroke="#cbd5e1"/>
    <text x="95" y="151" text-anchor="middle" font-weight="bold" fill="#334155">Thread-B</text>
    <text x="95" y="170" text-anchor="middle" font-size="12" fill="#64748b">logger.info(...)</text>
    <rect x="20" y="190" width="150" height="52" rx="6" fill="#f1f5f9" stroke="#cbd5e1"/>
    <text x="95" y="211" text-anchor="middle" font-weight="bold" fill="#334155">Thread-C</text>
    <text x="95" y="230" text-anchor="middle" font-size="12" fill="#64748b">logger.error(...)</text>

    <rect x="220" y="120" width="180" height="26" rx="6" fill="#3b82f6"/>
    <text x="310" y="137" text-anchor="middle" font-weight="bold" fill="#ffffff">Logger (client API)</text>
    <rect x="220" y="146" width="180" height="79" rx="6" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="232" y="166" font-weight="bold" fill="#374151">level enabled?</text>
    <text x="232" y="184" font-size="12.5" fill="#b91c1c">no → return (cheap)</text>
    <text x="232" y="202" font-size="12.5" fill="#1e3a8a">yes → LogRecord (immutable)</text>

    <rect x="450" y="135" width="170" height="26" rx="6" fill="#f59e0b"/>
    <text x="535" y="152" text-anchor="middle" font-weight="bold" fill="#ffffff">BlockingQueue</text>
    <rect x="450" y="161" width="170" height="124" rx="6" fill="#fffbeb" stroke="#fde68a"/>
    <text x="462" y="180" font-size="12.5" font-weight="bold" fill="#92400e">bounded · capacity 1024</text>
    <rect x="462" y="192" width="46" height="22" rx="4" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="485" y="207" text-anchor="middle" font-size="12" fill="#92400e">rec1</text>
    <rect x="512" y="192" width="46" height="22" rx="4" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="535" y="207" text-anchor="middle" font-size="12" fill="#92400e">rec2</text>
    <rect x="462" y="218" width="46" height="22" rx="4" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="485" y="233" text-anchor="middle" font-size="12" fill="#92400e">rec3</text>
    <rect x="512" y="218" width="46" height="22" rx="4" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="535" y="233" text-anchor="middle" font-size="12" fill="#92400e">…</text>
    <text x="462" y="258" font-size="12" fill="#92400e">offer: drop if full,</text>
    <text x="462" y="274" font-size="12" fill="#92400e">never stall a request</text>

    <rect x="690" y="150" width="150" height="46" rx="6" fill="#ede9fe" stroke="#8b5cf6"/>
    <text x="765" y="171" text-anchor="middle" font-weight="bold" fill="#4c1d95">log-writer thread</text>
    <text x="765" y="188" text-anchor="middle" font-size="12" fill="#6d28d9">single consumer</text>

    <rect x="690" y="285" width="180" height="46" rx="6" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="702" y="306" font-weight="bold" fill="#334155">ConsoleAppender</text>
    <text x="702" y="323" font-size="12" fill="#92400e">min level: WARN</text>
    <rect x="690" y="360" width="180" height="46" rx="6" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="702" y="381" font-weight="bold" fill="#334155">FileAppender</text>
    <text x="702" y="398" font-size="12" fill="#92400e">min level: DEBUG</text>
  </g>

  <g font-size="12.5" fill="#475569">
    <text x="423" y="152" text-anchor="middle">offer</text>
    <text x="648" y="215">take</text>
    <text x="770" y="240">per-sink level filter,</text>
    <text x="770" y="256">then Formatter → append</text>
  </g>

</svg>
```

The early return when the level is disabled is the performance story. A call to `logger.debug(...)` under an INFO config returns before building a `LogRecord`, so the hot path never pays for string formatting it will not use. That is why the level check must happen at the client boundary, not at the sink.

## Design Patterns Used

Three patterns genuinely fit, and each one maps to a real seam. `Appender` is a Strategy: the logger does not care whether the sink is a file, stdout, or a remote collector, it asks the strategy to write. `Formatter` is the same idea one level down. `LoggerConfig` with per-appender levels is a Chain of Responsibility in miniature: a record passes a filter at the logger, then another filter at each appender, and the first rejection stops it. Each pattern is doing actual filtering work, not wearing a costume. The one pattern to resist is the Singleton logger registry. A registry is fine, but a `getInstance()` Singleton that is also a `static` holder of all state makes testing impossible. Keep the registry a plain map and allow construction; the pattern that is genuinely wrong here is the one everyone reaches for.

## Handling Edge Cases / Concurrency

This is the case study where the concurrency section is the whole point. Three edges matter. First, thread safety of the record itself: solved by making `LogRecord` immutable, so the producer hands a frozen snapshot to the queue and no two threads ever share mutable state. Second, queue overflow: the bounded queue with `offer` means a burst of logging drops records instead of stalling requests, and that is a declared policy. Third, shutdown: when the application exits, the writer thread may hold records that were never written. The fix is a graceful `flush()` that drains the queue and closes appenders before the process ends, and a `shutdownNow` on the executor. Naming the shutdown path unprompted is a senior signal, because it is the edge that turns a working demo into a production system that does not lose the last hundred lines at every restart.

## Common Mistakes

The most common mistake is synchronous appenders written directly into `Logger.log`. File writes in the request path turn every log statement into an I/O stall, and the interviewer's "why does this not block" question has no answer. The queue is not decoration, it is the requirement.

The second mistake is the shared `SimpleDateFormat` or `StringBuilder` in the formatter. Formatters run on the single writer thread in this design, so it is safe here, but candidates who write their formatter to be thread-safe when it does not need to be, or who use a shared mutable formatter that is not thread-safe when it does need to be, reveal that they never decided who runs the formatter. Decide the threading model first, then write the formatter for it.

The third mistake is building the log message eagerly. `logger.info("order " + id + " shipped")` is fine, but `logger.info(buildExpensiveString())` runs the expensive builder even when the level is disabled. The level check must precede record construction, which is exactly why the check lives in `Logger.log` before `new LogRecord`.

## Interview Perspective

A weak answer is a static `Logger.log(String)` that appends to a file directly. There is no level, no sink abstraction, no queue, and the "two threads log at the same moment" question produces either corruption or a lock around every write. Nothing in the design can be extended, and nothing can be tested.

A strong answer names the three pieces in order: "the client sees only Logger, records flow through a queue to a single writer thread, and appenders and formatters are the seams." The follow-ups then answer themselves. "How do you add a JSON formatter" (new Formatter implementation, zero call-site changes). "How do you add a Kafka sink" (new Appender implementation). "What happens when the disk is full" (the appender swallows the error, the application survives, which is the degrade-gracefully requirement). "How do you avoid losing logs on shutdown" (flush and close, which the strong candidate already designed). The strongest candidates volunteer the bounded-queue policy decision and the shutdown flush without prompting, because they have run the walkthrough from a thread crash to process exit.

## Knowledge Check

1. A request thread calls `logger.info(...)` while the file appender is blocked on a slow disk. Trace the call through the dispatcher and explain why the request thread does not wait, then explain what eventually happens to the record and to the queue under sustained load.
2. The config enables DEBUG for a package but only WARN for the console appender. Walk a DEBUG record through the two filters and state which appender, if any, receives it.
3. The process receives a shutdown signal while the queue holds 300 records. Describe the minimal graceful shutdown path that writes those records, and what the writer thread must do before the process exits.

## Key Takeaways

- The client contract is one line: `logger.info(...)`. Everything else must be able to change beneath it.
- Producers enqueue, one writer consumes. That queue is the answer to every "does this block" question.
- Level checks happen at the client boundary, before record construction, or the hot path pays for formatting it will never use.
- Appender and Formatter are the two seams. New sinks and new shapes are new implementations, never call-site edits.
- Bounded queue and graceful shutdown are policy decisions. Make them, declare them, defend them.

## What's Next

The logging framework taught you the producer-consumer core and two clean seams. The inventory management system keeps the queue intuition at the margins but moves the center of gravity back to data, with the added weight of concurrent stock updates that can corrupt a business in a way a dropped log line never could.

---

This article explains how to design a logging framework around a producer-consumer queue where one writer thread drains immutable records. Its point of view is that the level check must happen before formatting, or the hot path pays for work it never uses.
