# Design a Notification System: Internal Class Orchestration

## Learning Objectives

- Learn to design a system whose value is in the orchestration: how one request flows through a pipeline of channels, templates, and delivery attempts without any single class holding the whole flow.
- Model the notification as a data record with a lifecycle, and the channels as interchangeable processors with different capabilities and failure profiles.
- Place the retry, dedupe, and rate-limit logic where they do not duplicate each other, which is the hardest part of any async pipeline.

## Introduction

Every other system in this book ends with "and then the user sees the result." The notification system starts there. The result is an email, an SMS, or a push to a device, and the system's job is to get that message from a business event to a delivery attempt reliably enough that the user actually receives it. The interesting design is not any single class, it is the orchestration: an order is shipped, some service publishes that fact, the notification pipeline decides that this customer wants an email and a push but not an SMS, renders each channel's message from a shared template, and delivers each one without the email provider's rate limits flattening a batch of fifty thousand. Interviewers ask this because it is a pure orchestration problem, and because the class structure of a good pipeline, queue in front, dispatcher in the middle, channels at the end, shows up in every async system they hire for.

## Requirements Gathering

Functional requirements:

- A business event (order shipped, payment failed) triggers notifications to a user.
- Each user has channel preferences; each notification type has a channel template.
- The system renders the right message for each channel, queues the deliveries, and sends them through the channel providers.
- Deliveries that fail are retried with backoff, and duplicate notifications for the same event are suppressed.

Non-functional requirements:

- A spike in events must not overwhelm any downstream provider; the pipeline must absorb bursts.
- The system must degrade per channel: an email outage must not block SMS or push.

Assumptions to state out loud: no in-app notification inbox or read receipts, no push token lifecycle beyond a simple registry, delivery is best-effort with a bounded number of retries rather than exactly-once, and templates are static strings with placeholder substitution, not a full templating language. Cut read receipts and cut the in-app feed. The interviewer wants the orchestration, and the feed is a whole storage system bolted on.

## Identifying Core Entities

The entity list is a pipeline, and each class is one stage with one job.

| Entity | One-line responsibility |
| --- | --- |
| `NotificationEvent` | The external fact that triggers notifications. |
| `Notification` | The rendered, per-channel message record with its lifecycle status. |
| `NotificationType` | The logical kind of message, mapping to templates and channel rules. |
| `TemplateRenderer` | Fills a channel template from event data. |
| `Channel` | An abstract delivery processor; email, SMS, and push implementations. |
| `NotificationQueue` | The buffer that decouples event handling from delivery. |
| `NotificationDispatcher` | The worker that takes queued notifications and routes them to channels. |
| `UserPreferenceService` | Answers which channels a user wants for which notification type. |

The pipeline reads like a sentence: event goes in, preferences decide channels, templates render messages, the queue absorbs the load, the dispatcher routes, and the channels deliver. Each class is one word of that sentence.

## Class Design

Start with the record that flows through the pipeline. `Notification` is the unit of work: which user, which type, which channel, the rendered message, and the delivery state. Its state machine, QUEUED, SENDING, SENT, FAILED, is what the dispatcher and the retry logic both read.

```java
public class Notification {
    public enum Status { QUEUED, SENDING, SENT, FAILED }

    private final String notificationId;
    private final String userId;
    private final NotificationType type;
    private final Channel channel;
    private final String renderedMessage;
    private Status status = Status.QUEUED;
    private int attempts = 0;

    public Notification(String notificationId, String userId, NotificationType type,
                        Channel channel, String renderedMessage) {
        this.notificationId = notificationId;
        this.userId = userId;
        this.type = type;
        this.channel = channel;
        this.renderedMessage = renderedMessage;
    }

    public void markSending() { status = Status.SENDING; attempts++; }
    public void markSent() { status = Status.SENT; }
    public void markFailed() { status = Status.FAILED; }
    public int getAttempts() { return attempts; }
    public Status getStatus() { return status; }
    public String getUserId() { return userId; }
    public Channel getChannel() { return channel; }
    public String getRenderedMessage() { return renderedMessage; }
}
```

The channel is the strategy that makes each delivery path replaceable. `Channel` carries the delivery attempt and its own rate-limiting and failure behavior, so the email channel can throttle itself without the dispatcher knowing anything about email.

```java
public abstract class Channel {
    private final String name;
    private final int maxAttempts;

    protected Channel(String name, int maxAttempts) {
        this.name = name;
        this.maxAttempts = maxAttempts;
    }

    public String getName() { return name; }
    public int getMaxAttempts() { return maxAttempts; }

    public abstract boolean send(Notification notification);

    public boolean retryable(Notification notification) {
        return notification.getStatus() == Notification.Status.FAILED
                && notification.getAttempts() < maxAttempts;
    }
}

public class EmailChannel extends Channel {
    private final EmailProvider provider;

    public EmailChannel(EmailProvider provider) {
        super("email", 5);
        this.provider = provider;
    }

    public boolean send(Notification notification) {
        try {
            provider.send(notification.getUserId(), notification.getRenderedMessage());
            return true;
        } catch (ProviderException e) {
            return false;
        }
    }
}
```

The template renderer is the point where data becomes a message. The design decision is that rendering is channel-aware: the same event renders differently for email, which has room for a header and a footer, than for SMS, which has 160 characters and a cut-down message.

```java
public class TemplateRenderer {
    private final Map<String, String> templates = new HashMap<>();

    public void register(NotificationType type, Channel channel, String template) {
        templates.put(type.name() + ":" + channel.getName(), template);
    }

    public String render(NotificationEvent event, Channel channel) {
        String template = templates.get(event.getType().name() + ":" + channel.getName());
        if (template == null) {
            throw new IllegalArgumentException("No template for " + event.getType() + " on " + channel.getName());
        }
        String rendered = template;
        for (Map.Entry<String, String> e : event.getData().entrySet()) {
            rendered = rendered.replace("{{" + e.getKey() + "}}", e.getValue());
        }
        return rendered;
    }
}
```

The queue and dispatcher are the orchestration. The queue is a `BlockingQueue` (or, in production, Kafka or SQS), and the dispatcher is the worker pool that consumes it. The dispatcher is where the system earns its "does not flatten a provider" requirement: it hands each notification to its channel, and the channel's own retry and rate behavior does the throttling.

```java
public class NotificationDispatcher {
    private final BlockingQueue<Notification> queue = new LinkedBlockingQueue<>(10_000);
    private final ExecutorService workers = Executors.newFixedThreadPool(8);
    private final List<Channel> channels;

    public NotificationDispatcher(List<Channel> channels) {
        this.channels = channels;
    }

    public void enqueue(Notification notification) {
        queue.offer(notification);
    }

    public void start() {
        for (int i = 0; i < 8; i++) {
            workers.submit(this::drain);
        }
    }

    private void drain() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                Notification notification = queue.take();
                process(notification);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    private void process(Notification notification) {
        notification.markSending();
        boolean delivered = notification.getChannel().send(notification);
        if (delivered) {
            notification.markSent();
        } else if (notification.getChannel().retryable(notification)) {
            scheduleRetry(notification);
        } else {
            notification.markFailed();
        }
    }

    private void scheduleRetry(Notification notification) {
        long backoffMillis = (long) Math.pow(2, notification.getAttempts()) * 1000;
        workers.submit(() -> {
            try {
                Thread.sleep(backoffMillis);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            process(notification);
        });
    }
}
```

`NotificationOrchestrator` is the front door. It is the one place where an external event becomes a set of notifications, and it is where the per-user channel preferences and the dedupe check belong, because both are decisions made before the queue, not after.

```java
public class NotificationOrchestrator {
    private final UserPreferenceService preferences;
    private final TemplateRenderer renderer;
    private final NotificationDispatcher dispatcher;
    private final Set<String> dedupeKeys = ConcurrentHashMap.newKeySet();

    public void handle(NotificationEvent event) {
        if (!dedupeKeys.add(event.getDedupeKey())) {
            return; // already handled this event
        }
        for (Channel channel : preferences.channelsFor(event.getUser(), event.getType())) {
            String message = renderer.render(event, channel);
            Notification notification = new Notification(
                    UUID.randomUUID().toString(), event.getUser(),
                    event.getType(), channel, message);
            dispatcher.enqueue(notification);
        }
    }
}
```

Diagram: the pipeline. An event is deduped, channel preferences are applied, and each channel's message is rendered before the queue. The dispatcher hands off to channels that own their own retry and rate behavior.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 960 380" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah4" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="960" height="380" fill="#ffffff"/>

  <text x="480" y="28" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">The notification pipeline</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah4)">
    <line x1="150" y1="240" x2="206" y2="240"/>
    <line x1="400" y1="240" x2="446" y2="240"/>
    <line x1="570" y1="240" x2="606" y2="240"/>
    <line x1="740" y1="240" x2="790" y2="124"/>
    <line x1="740" y1="240" x2="790" y2="224"/>
    <line x1="740" y1="240" x2="790" y2="324"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="20" y="205" width="130" height="70" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="85" y="235" text-anchor="middle" font-weight="bold" fill="#334155">Notification</text>
    <text x="85" y="253" text-anchor="middle" font-weight="bold" fill="#334155">Event</text>
    <text x="85" y="269" text-anchor="middle" font-size="12" fill="#64748b">order shipped</text>

    <rect x="210" y="170" width="190" height="26" rx="6" fill="#3b82f6"/>
    <text x="305" y="187" text-anchor="middle" font-weight="bold" fill="#ffffff">NotificationOrchestrator</text>
    <rect x="210" y="196" width="190" height="114" rx="6" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="222" y="218">1. dedupe check (front door)</text>
    <text x="222" y="240">2. preferences → channels</text>
    <text x="222" y="262">3. render per channel</text>
    <text x="222" y="284" font-size="12" fill="#b45309">from shared template</text>

    <rect x="450" y="205" width="120" height="70" rx="8" fill="#fffbeb" stroke="#f59e0b"/>
    <text x="510" y="235" text-anchor="middle" font-weight="bold" fill="#92400e">Notification</text>
    <text x="510" y="253" text-anchor="middle" font-weight="bold" fill="#92400e">Queue</text>
    <text x="510" y="269" text-anchor="middle" font-size="12" fill="#b45309">bounded buffer</text>

    <rect x="610" y="205" width="130" height="70" rx="8" fill="#ede9fe" stroke="#8b5cf6"/>
    <text x="675" y="235" text-anchor="middle" font-weight="bold" fill="#4c1d95">Notification</text>
    <text x="675" y="253" text-anchor="middle" font-weight="bold" fill="#4c1d95">Dispatcher</text>
    <text x="675" y="269" text-anchor="middle" font-size="12" fill="#6d28d9">worker pool ×8</text>

    <rect x="790" y="100" width="160" height="48" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="870" y="123" text-anchor="middle" font-weight="bold" fill="#1e3a8a">EmailChannel</text>
    <text x="870" y="140" text-anchor="middle" font-size="12" fill="#1e40af">maxAttempts 5 · rate-limited</text>
    <rect x="790" y="200" width="160" height="48" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="870" y="223" text-anchor="middle" font-weight="bold" fill="#14532d">SmsChannel</text>
    <text x="870" y="240" text-anchor="middle" font-size="12" fill="#15803d">160-char message</text>
    <rect x="790" y="300" width="160" height="48" rx="8" fill="#ede9fe" stroke="#8b5cf6"/>
    <text x="870" y="323" text-anchor="middle" font-weight="bold" fill="#4c1d95">PushChannel</text>
    <text x="870" y="340" text-anchor="middle" font-size="12" fill="#6d28d9">device registry</text>
  </g>

  <g font-size="12.5" fill="#475569" text-anchor="middle">
    <text x="178" y="228">handle(event)</text>
    <text x="423" y="228">enqueue</text>
    <text x="588" y="228">take</text>
    <text x="772" y="158">send</text>
    <text x="772" y="180">send</text>
  </g>

</svg>
```

The orchestration in one sentence: dedupe first, prefer channels, render per channel, enqueue, and the dispatcher and the channels take it from there.

## Design Patterns Used

Three patterns earn their keep here, and each one is doing real work. The `Channel` hierarchy is a Strategy, the per-channel delivery behavior that the rest of the system does not need to understand. The `TemplateRenderer` is a Template Method in spirit, the fixed skeleton of "replace placeholders" with the channel-specific template as the variation. And the queue-plus-dispatcher is the classic producer-consumer pattern from the logging chapter, scaled up. That is three patterns with three reasons, which is rare, and the discipline is the same as everywhere: name what each one is for, and be ready to say what breaks if you remove it. If the dispatcher were a giant `if (channel == email) sendEmail(...)`, removing any pattern becomes a rewrite.

## Handling Edge Cases / Concurrency

The orchestration edge cases are where the design lives. Dedupe: the same business event must not produce the same notification twice, which is why the dedupe check is at the orchestrator's front door, before rendering, with the event's dedupe key derived from the event type, the user, and a stable event ID. In production the dedupe store is Redis with an expiry, because a memory set grows forever; name that.

The retry loop: a notification that fails must not spin. The bounded attempts and the exponential backoff are on the channel, and the retry is a re-enqueue with sleep, not a blocking loop inside the worker. The dead-letter path, a notification that exhausted its attempts, must land somewhere visible, which in production is a DLQ topic; the design has `markFailed` as the terminal state, and the honest answer is that a failed notification is logged and surfaced, not silently dropped.

The concurrency: multiple workers pulling from one queue means the same notification must never be processed by two workers at once, which the take-then-mark-sending ordering almost guarantees but does not prove. The production answer is a consumer group with per-key ordering, or a lock on the notification ID. Name it.

## Common Mistakes

The most common mistake is the god-orchestrator: one class that renders, preferences-checks, sends, and retries, all synchronously. The interviewer asks "what happens when fifty thousand emails come at once" and the answer is "the orchestrator blocks," which is the whole requirement violated. The queue is not decoration, it is the absorption mechanism.

The second mistake is the channel switch. `switch (channel) { case EMAIL: ... case SMS: ... }` in the dispatcher, so every new channel is an edit to the dispatcher and a new branch in the retry logic. The channel hierarchy is the extension point, and adding push should be one class.

The third mistake is retry logic in three places. The orchestrator retries, the dispatcher retries, and the channel retries, each with its own counter. The retry is a property of the channel's delivery profile, and it lives on the channel. One source of truth for how many attempts a channel gets.

## Interview Perspective

A weak answer is a `NotificationService` with a `sendEmail`, `sendSms`, and `sendPush` method, called from the order service directly. There is no queue, no channel abstraction, no template, no retry. The interviewer asks "the email provider is down, what happens to the SMS" and the answer is "well, they're both in sendEmail" or silence.

A strong answer names the pipeline in one sentence: "event in, preferences pick channels, templates render per channel, the queue absorbs the burst, the dispatcher hands off to channels that own their own retry and rate behavior." Follow-ups to expect: "what if the push provider rate-limits" (the channel throttles itself, which is why rate behavior lives on the channel), "how do you avoid duplicate notifications when the order service retries the event" (the dedupe key at the orchestrator's front door), "how do you add a new channel, say WhatsApp" (one channel class, one template registration, zero edits to the pipeline). The strongest candidates volunteer the dead-letter path unprompted, because they have watched failed notifications vanish into a void in production and know the difference.

## Knowledge Check

1. A business event triggers email and push for one user, and the email provider is down for two hours. Trace the lifecycle of the email notification and explain why the push notification is not delayed by it.
2. The order service retries its publish call, so the same event arrives twice at the orchestrator. Which method and which data prevent the second arrival from producing duplicate notifications?
3. The push channel's provider starts rejecting requests. Walk through where the retry counter and the backoff live, and describe what happens to a notification that exhausts its attempts.

## Key Takeaways

- The pipeline is the design: dedupe, prefer, render, enqueue, dispatch, deliver. One class per stage, no god class.
- Channels own their delivery profile: attempts, backoff, rate. The dispatcher knows none of it.
- The queue is the burst absorber. Without it, the system is synchronous and the requirement is already dead.
- Dedupe lives at the front door, before anything is rendered.
- Retry has one owner, the channel, and a terminal state that does not silently drop.

## What's Next

The notification system was a pipeline with a queue in the middle. The rate limiter removes the pipelines and the channels and keeps only the gate, and the design becomes a different question entirely: not how to deliver, but how to say no, with the count and the window as the only variables.

---

This article explains how to design a notification system as a pipeline where events become per-channel messages through a queue and a dispatcher. Its point of view is that channels must own their retry behavior, or a burst of events flattens the whole system.
