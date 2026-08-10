# Design a Job Scheduler

## Learning Objectives

- Model a job as a data record with a trigger, and the trigger as the thing that decides when the job enters the runnable set.
- Build the scheduler core, a priority queue keyed by next run time, and see why the heap is not an optimization but the algorithm.
- Handle the two hard edges, a job that must not run twice and a worker that dies mid-job, and know which machinery solves each.

## Introduction

The job scheduler is the case study where the data structure is the design. A rate limiter is a counter and a clock; a scheduler is a queue and a clock. Jobs come in with a time to run, now, in ten minutes, at 3 AM daily, and the system's job is to run each one exactly when it is due, exactly once, without losing it when a worker dies. The interview is about three things: how you represent the "when," how you find the next job to run, and how you survive the fact that workers are machines that crash. Every job-scheduling product in the real world, from Quartz to Celery to a cron replacement, is the same heap, the same trigger model, and the same at-least-once-with-idempotency concession wearing a different name.

## Requirements Gathering

Functional requirements:

- A job carries a payload and a trigger: one-time at a time, delayed by a duration, or recurring on an interval.
- The scheduler makes a job runnable at its due time and dispatches it to a worker.
- A running job has a state: scheduled, due, running, completed, failed.
- Failed jobs retry a bounded number of times with backoff, then land in a dead-letter state.
- A job can be cancelled before it runs.

Non-functional requirements:

- Finding the next due job must be fast regardless of how many jobs are scheduled, because the check runs constantly.
- The system must not run the same job twice, even if a worker crashes mid-job and the job is retried.

Assumptions to state out loud: no cron expressions, an interval trigger is enough, no job priorities beyond the time itself, no distributed coordination beyond the single-leader model, and workers run one job at a time. Cut cron parsing, it is a parser and a library problem, not a design problem. The interviewer wants the heap and the lifecycle.

## Identifying Core Entities

The entity list is a queue with a clock and a lifecycle.

| Entity | One-line responsibility |
| --- | --- |
| `Job` | The unit of work: ID, payload, type, and lifecycle state. |
| `Trigger` | The rule that produces the job's next run time. |
| `JobQueue` | The priority structure of due and upcoming jobs, keyed by next run time. |
| `Scheduler` | The loop that promotes due jobs and dispatches them. |
| `WorkerPool` | The executor that runs jobs. |
| `JobStore` | The durable record of every job and its state. |

The entity carrying the design is `JobQueue`, and it should be a heap. The trigger is the thing that makes the heap meaningful, because without a "next time" the queue has nothing to sort by.

## Class Design

Start with `Trigger`. It is an interface with two implementations, one-time and interval, and its contract is the single method that the whole scheduler runs on: given a reference time, what is the next run time?

```java
public interface Trigger {
    Instant nextRunTime(Instant now);
}

public class OneTimeTrigger implements Trigger {
    private final Instant runAt;

    public OneTimeTrigger(Instant runAt) { this.runAt = runAt; }

    public Instant nextRunTime(Instant now) {
        return runAt.isAfter(now) ? runAt : null;
    }
}

public class IntervalTrigger implements Trigger {
    private final Duration interval;

    public IntervalTrigger(Duration interval) { this.interval = interval; }

    public Instant nextRunTime(Instant now) {
        return now.plus(interval);
    }
}
```

`Job` is the unit of work, carrying its trigger, its state, and its retry counter. The lifecycle methods are the guard rails, same shape as the booking and the order from earlier chapters.

```java
public class Job {
    public enum Status { SCHEDULED, DUE, RUNNING, COMPLETED, FAILED, CANCELLED }

    private final String jobId;
    private final String payload;
    private final Trigger trigger;
    private Instant nextRunAt;
    private Status status = Status.SCHEDULED;
    private int attempts = 0;

    public Job(String jobId, String payload, Trigger trigger, Instant nextRunAt) {
        this.jobId = jobId;
        this.payload = payload;
        this.trigger = trigger;
        this.nextRunAt = nextRunAt;
    }

    public boolean markRunning() {
        if (status != Status.DUE) return false;
        status = Status.RUNNING;
        attempts++;
        return true;
    }

    public void markCompleted() { status = Status.COMPLETED; }
    public void markFailed() { status = Status.FAILED; }
    public void markCancelled() { status = Status.CANCELLED; }

    public Instant getNextRunAt() { return nextRunAt; }
    public void setNextRunAt(Instant nextRunAt) { this.nextRunAt = nextRunAt; }
    public int getAttempts() { return attempts; }
    public Status getStatus() { return status; }
    public String getJobId() { return jobId; }
    public String getPayload() { return payload; }
}
```

`JobQueue` is the heap. A `PriorityQueue` keyed by `nextRunAt` gives O(log n) insert and O(1) peek of the next job, which is the property the scheduler's constant checking depends on. The queue is the algorithm, and the algorithm is the heap.

```java
public class JobQueue {
    private final PriorityQueue<Job> heap = new PriorityQueue<>(
            Comparator.comparing(Job::getNextRunAt));

    public synchronized void schedule(Job job) { heap.add(job); }

    public synchronized Optional<Job> nextDue(Instant now) {
        Job top = heap.peek();
        if (top != null && !top.getNextRunAt().isAfter(now)) {
            return Optional.of(heap.poll());
        }
        return Optional.empty();
    }
}
```

`Scheduler` is the loop. It alternates between two actions: promote due jobs to the workers, and wait until the next job is due. The waiting is what keeps the scheduler from spinning: it sleeps until the head of the heap, not in a busy loop.

```java
public class Scheduler {
    private final JobQueue queue;
    private final WorkerPool workers;
    private final JobStore store;

    public Scheduler(JobQueue queue, WorkerPool workers, JobStore store) {
        this.queue = queue;
        this.workers = workers;
        this.store = store;
    }

    public void runLoop() {
        while (!Thread.currentThread().isInterrupted()) {
            Optional<Job> due = queue.nextDue(Instant.now());
            if (due.isEmpty()) {
                sleepUntilHead();
                continue;
            }
            Job job = due.get();
            if (!job.markRunning()) {
                continue;
            }
            store.save(job);
            workers.execute(job, this::onComplete);
        }
    }

    private void onComplete(Job job, boolean success) {
        if (success) {
            job.markCompleted();
            store.save(job);
            rescheduleIfRecurring(job);
        } else if (job.getAttempts() >= 3) {
            job.markFailed();
            store.save(job);
        } else {
            scheduleRetry(job);
        }
    }

    private void rescheduleIfRecurring(Job job) {
        Instant next = job.getTrigger().nextRunTime(Instant.now());
        if (next != null) {
            job.setNextRunAt(next);
            job.markScheduled();
            queue.schedule(job);
            store.save(job);
        }
    }
}
```

Diagram: the scheduler's two ideas in one view. Top, the `JobQueue` as a min-heap keyed by next run time, where the head is always the next due job. Bottom, the job lifecycle, including the two recovery paths for a worker that fails or crashes.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 880 560" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#6b7280"/>
    </marker>
  </defs>
  <rect width="880" height="560" fill="#ffffff"/>

  <text x="440" y="30" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">JobQueue = a min-heap keyed by nextRunAt</text>

  <g stroke="#cbd5e1" stroke-width="2">
    <line x1="450" y1="99" x2="355" y2="148"/>
    <line x1="450" y1="99" x2="545" y2="148"/>
    <line x1="290" y1="190" x2="185" y2="218"/>
    <line x1="290" y1="190" x2="395" y2="218"/>
    <line x1="610" y1="190" x2="525" y2="218"/>
    <line x1="610" y1="190" x2="705" y2="218"/>
  </g>

  <g font-size="13" font-weight="bold" text-anchor="middle">
    <rect x="385" y="78" width="130" height="42" rx="8" fill="#dcfce7" stroke="#16a34a" stroke-width="2"/>
    <text x="450" y="104" fill="#14532d">J4 · 10:00</text>
    <text x="450" y="66" fill="#16a34a" font-size="12">head — due first</text>
    <rect x="225" y="148" width="130" height="42" rx="8" fill="#f3f4f6" stroke="#d1d5db"/>
    <text x="290" y="174" fill="#374151">J1 · 10:15</text>
    <rect x="545" y="148" width="130" height="42" rx="8" fill="#f3f4f6" stroke="#d1d5db"/>
    <text x="610" y="174" fill="#374151">J7 · 10:30</text>
    <rect x="120" y="218" width="130" height="42" rx="8" fill="#f3f4f6" stroke="#d1d5db"/>
    <text x="185" y="244" fill="#374151">J3 · 11:00</text>
    <rect x="330" y="218" width="130" height="42" rx="8" fill="#f3f4f6" stroke="#d1d5db"/>
    <text x="395" y="244" fill="#374151">J9 · 11:05</text>
    <rect x="460" y="218" width="130" height="42" rx="8" fill="#f3f4f6" stroke="#d1d5db"/>
    <text x="525" y="244" fill="#374151">J2 · 11:30</text>
    <rect x="640" y="218" width="130" height="42" rx="8" fill="#f3f4f6" stroke="#d1d5db"/>
    <text x="705" y="244" fill="#374151">J5 · 12:00</text>
  </g>

  <text x="440" y="298" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">Job lifecycle</text>

  <g font-size="13" font-weight="bold" text-anchor="middle">
    <rect x="40" y="340" width="150" height="46" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="115" y="367" fill="#1e3a8a">SCHEDULED</text>
    <rect x="240" y="340" width="150" height="46" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="315" y="367" fill="#92400e">DUE</text>
    <rect x="440" y="340" width="150" height="46" rx="8" fill="#ede9fe" stroke="#8b5cf6"/>
    <text x="515" y="367" fill="#4c1d95">RUNNING</text>
    <rect x="650" y="340" width="150" height="46" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="725" y="367" fill="#14532d">COMPLETED</text>
    <rect x="40" y="252" width="150" height="40" rx="8" fill="#f3f4f6" stroke="#9ca3af"/>
    <text x="115" y="276" fill="#4b5563">CANCELLED</text>
    <rect x="650" y="252" width="150" height="40" rx="8" fill="#fee2e2" stroke="#dc2626"/>
    <text x="725" y="276" fill="#7f1d1d">FAILED</text>
  </g>

  <g stroke="#6b7280" stroke-width="2" fill="none" marker-end="url(#ah)">
    <line x1="190" y1="363" x2="236" y2="363"/>
    <line x1="390" y1="363" x2="436" y2="363"/>
    <line x1="590" y1="363" x2="646" y2="363"/>
    <line x1="115" y1="340" x2="115" y2="296"/>
    <polyline points="515,386 515,430 315,430 315,390"/>
    <polyline points="590,386 590,478 115,478 115,390"/>
  </g>
  <g stroke="#dc2626" stroke-width="2" stroke-dasharray="6 4" fill="none" marker-end="url(#ah)">
    <line x1="515" y1="340" x2="700" y2="296"/>
  </g>

  <g font-size="12.5" fill="#374151" text-anchor="middle">
    <text x="213" y="352">due time reached</text>
    <text x="413" y="352">dispatch</text>
    <text x="618" y="352">success</text>
    <text x="84" y="318">cancel</text>
    <text x="628" y="318" fill="#b91c1c">fail, attempts ≥ 3 → dead-letter</text>
    <text x="420" y="448">lease expires (worker crash) → back to DUE</text>
    <text x="315" y="472">fail, attempts &lt; 3 → retry with backoff</text>
  </g>
</svg>
```

The worker pool is a fixed-size executor. The important design detail is that the worker's completion callback, not the job itself, owns the retry and reschedule decisions, so the scheduler's lifecycle logic stays in one place.

## Design Patterns Used

The pattern here is the Command pattern, and for once it is not overkill: a `Job` genuinely is a command, a self-contained unit of work with a payload and a trigger that a scheduler can queue, persist, and dispatch without knowing what the work is. Naming that is correct. The `Trigger` is a Strategy in miniature, one interface, two implementations, and the retry policy is a small policy object. The producer-consumer shape, scheduler produces work, workers consume it, is the same one from the logging and notification chapters, which is the point: this book keeps building the same three shapes, and a candidate who recognizes the pattern across case studies is the candidate who is actually learning.

## Handling Edge Cases / Concurrency

The scheduler has two hard edges, and both are about workers failing. The first is the crash mid-job. A worker dies after picking up a job but before completing it; the job is RUNNING forever. In production this is handled by a lease: the worker holds a lease on the job with a timeout, and when the lease expires, the job returns to DUE. The design's honest version is that the scheduler, or a watchdog, scans for RUNNING jobs whose lease lapsed and re-queues them. Name the lease mechanism, because "what if a worker dies" is the first question the interviewer asks.

The second edge is the exactly-once problem, and the honest answer is that the scheduler provides at-least-once and the job itself provides idempotency. A job retried after a crash runs twice unless its payload is idempotent, and the design's job is to make the second run harmless by design, not to prevent it. State that sentence and you have answered the deepest question in the case study.

The concurrency inside the scheduler: a single scheduler thread owns the heap, so the queue's `synchronized` methods are enough. The multi-scheduler story is a leader election, one scheduler owns the queue at a time, which is the distributed version of the same heap, and it is the "how do you scale" follow-up.

## Common Mistakes

The most common mistake is a linear scan for due jobs. The candidate draws a `List<Job>` and a `for` loop that checks every job's due time on every tick. That design costs O(n) per check and the interviewer's "what if there are a million jobs" is unanswerable. The heap is not a refinement, it is the reason the scheduler can check constantly.

The second mistake is busy-waiting. A loop that checks the queue in a tight cycle burns a core checking jobs that are not due yet. The scheduler must sleep until the head of the heap is due, and the interviewer's "what does the scheduler do while waiting" has a wrong answer and a right one.

The third mistake is treating completion and failure as worker responsibilities. The worker that decides to retry, reschedule, or dead-letter means three workers make three different retry decisions for the same job type. The lifecycle logic belongs in the scheduler's callback, one place, one policy.

## Interview Perspective

A weak answer is a `Thread.sleep(1000)` loop over a list of jobs with an `if (dueNow)` check, and no heap, no lease, and no lifecycle states. The interviewer asks "what happens when a worker dies mid-job" and the job is lost or stuck RUNNING with no mechanism, and "how do you run a million scheduled jobs" has no answer.

A strong answer says "the heap is the algorithm, the trigger is the strategy that produces next-run times, the retry policy is the scheduler's callback, not the worker's, and the exactly-once answer is at-least-once with idempotent jobs." Follow-ups to expect: "what if a job takes longer than the interval" (the recurring reschedule computes from completion time, so the job does not overlap itself, or a `running` guard rejects the overlap, and the strong candidate picks one), "how do you scale to multiple schedulers" (leader election, one owner of the heap, which is the distributed version of the same structure), "how do you cancel a job" (mark CANCELLED in the store, and the heap ignores cancelled jobs when it polls them). The strongest candidates volunteer the lease timeout and the idempotency caveat without prompting, because they have debugged a job that ran twice at 3 AM.

## Knowledge Check

1. A scheduler holds a million scheduled jobs and checks for due work every few seconds. Explain why the heap makes this viable and the linear list does not, in terms of the operations each check performs.
2. A worker picks up a job and the process crashes. Walk through what the job's status is, what mechanism returns it to the queue, and what guarantee the system can and cannot make about the job running exactly once.
3. A recurring job's execution takes five minutes but its interval is three minutes. Describe what happens at the three-minute mark, and the two policy choices the scheduler can make about the overlap.

## Key Takeaways

- The priority heap keyed by next run time is the scheduler's algorithm. Everything else is the lifecycle around it.
- Sleep until the head of the heap is due. Busy-waiting is a bug.
- Retry, dead-letter, and reschedule decisions live in the scheduler's completion callback, not in the workers.
- Workers crash. The lease timeout returns stuck jobs to the queue, and at-least-once plus idempotent jobs is the honest guarantee.
- Recurring reschedules compute from completion, not from the scheduled time, or the job overlaps itself.

## What's Next

The job scheduler was a heap and a clock with a lifecycle around it. The real-time chat system keeps the async handoff but makes it bidirectional and immediate, and the design shifts from "when does this run" to "how does a message get from one socket to another without ever landing in a database on the way."

---

This article explains how to design a job scheduler around a priority heap of next run times and a lifecycle that recovers jobs from crashed workers. It argues the honest guarantee is at-least-once delivery with idempotent jobs, since exactly-once is a lie.
