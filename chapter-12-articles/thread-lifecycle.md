# Thread Lifecycle

## Learning Objectives

- Name the six thread states in Java and the transition that moves a thread into each one.
- Read a thread dump and explain, from the state names, what a stuck thread is actually waiting on.
- Distinguish BLOCKED from WAITING and the two timers that control them.

## Introduction

A thread is not a thing that runs; it is a thing that is sometimes running. Java models that with an explicit set of states, `NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, `TERMINATED`, and every tool in this chapter is a way of pushing a thread into one of them. Knowing the states is the difference between reading a thread dump and reading a riddle.

## Problem Statement

A server hangs. The request thread that used to answer is now idle, and nobody can say whether it is waiting for a lock, waiting for a notification, or just asleep for a while. The team guesses. They restart, it hangs again, and the eventual answer, a single thread stuck in `WAITING` on an object monitor because the code that was supposed to `notify()` never ran, takes two days. The six states exist precisely so you never have to guess: the state name tells you what the thread is blocked on, and that is the start of every real diagnosis.

## Core Concept

The states are a state machine the JVM drives, and each transition is caused by something concrete.

| State | Meaning | How you get there | How you leave |
|-------|---------|-------------------|---------------|
| NEW | constructed, not started | `new Thread(...)` | `start()` |
| RUNNABLE | ready or actually on a CPU | `start()`, returning from a wait | scheduling |
| BLOCKED | waiting to enter a `synchronized` block | a lock is held by someone else | the lock is released |
| WAITING | waiting indefinitely for a signal | `wait()`, `join()`, `park()` | `notify()`/`notifyAll()`, unpark |
| TIMED_WAITING | waiting with a deadline | `sleep()`, `wait(timeout)`, `join(timeout)` | timeout or signal |
| TERMINATED | done or threw | `run()` returns | nothing |

A thread in `RUNNABLE` may or may not be executing right now. That state conflates "on a core" and "in the ready queue", which is deliberate, because Java cannot cheaply tell you which. So a busy CPU-bound thread and a thread that keeps getting descheduled both read `RUNNABLE`, and that is fine, the state is about the ability to run, not about possessing a core.

The division that matters for debugging is `BLOCKED` versus `WAITING`. `BLOCKED` means the thread wants to enter a synchronized block and the monitor is taken. `WAITING` means the thread is parked until another thread calls a notifier. The two are different promises: one waits for a lock, the other waits for a wake-up. A thread dump that shows one stuck in `WAITING` on `Object#wait` is a thread that is waiting to be told something, and the fix is in the thread that should have told it.

Diagram: the six states and the transitions between them.

<svg width="860" height="420" viewBox="0 0 860 420" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="th" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#222"/>
    </marker>
  </defs>

  <ellipse cx="90" cy="60" rx="58" ry="30" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="90" y="65" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">NEW</text>

  <ellipse cx="300" cy="60" rx="70" ry="30" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="300" y="65" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">RUNNABLE</text>

  <ellipse cx="520" cy="60" rx="66" ry="30" fill="#f8dede" stroke="#962828" stroke-width="2"/>
  <text x="520" y="65" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">BLOCKED</text>

  <ellipse cx="220" cy="200" rx="64" ry="30" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="220" y="205" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">WAITING</text>

  <ellipse cx="430" cy="200" rx="92" ry="30" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="430" y="205" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">TIMED_WAITING</text>

  <ellipse cx="620" cy="200" rx="78" ry="30" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="620" y="205" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">TERMINATED</text>

  <path d="M 148 60 L 228 60" stroke="#222" stroke-width="2" fill="none" marker-end="url(#th)"/>
  <text x="188" y="50" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">start()</text>

  <path d="M 370 60 C 430 60 430 60 452 60" stroke="#222" stroke-width="2" fill="none" marker-end="url(#th)"/>
  <text x="412" y="50" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">lock taken</text>

  <path d="M 520 90 C 400 150 400 160 260 185" stroke="#962828" stroke-width="2" fill="none" marker-end="url(#th)"/>
  <text x="392" y="140" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">wait(), join()</text>

  <path d="M 520 90 C 520 150 460 165 460 185" stroke="#8a6d00" stroke-width="2" fill="none" marker-end="url(#th)"/>
  <text x="548" y="140" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">sleep()</text>

  <path d="M 368 185 C 368 140 340 110 330 92" stroke="#222" stroke-width="2" fill="none" marker-end="url(#th)"/>
  <text x="352" y="130" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">notify()</text>

  <path d="M 620 170 C 560 120 460 95 380 70" stroke="#222" stroke-width="2" fill="none" marker-end="url(#th)"/>
  <text x="560" y="100" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">lock released</text>

  <path d="M 300 90 C 300 140 260 160 245 172" stroke="#8a6d00" stroke-width="2" fill="none" marker-end="url(#th)"/>
  <text x="270" y="145" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">timeout</text>
</svg>

There are a few details in that diagram that deserve a second look. `BLOCKED` and the lock release arrow is the story of a thread that wants in and waits. The `notify()` arrow is the story of `WAITING`. The `sleep()` path is `TIMED_WAITING`, and note that `sleep` does not release any locks you hold, which makes it a common cause of a locked object staying locked. There is no direct state called "deadlock"; a deadlock in a dump appears as a cycle of threads, each `BLOCKED` on a monitor that the next one in the cycle holds.

## Real Production Usage

`Thread.currentThread().getState()` and, more usefully, a JVM thread dump (`jstack`, or the kill -3 path, or `Thread.dumpAllThreads`) are the tools. A healthy pool shows threads parked in `WAITING` on a pool condition, which is normal. A pool that has run out shows threads in `WAITING` on the pool's `take()` forever, which means the pool died or the tasks are stuck. Reading a dump well means asking, of each stuck thread, "is it waiting for a lock, a signal, or a timer?" and each state answers it directly.

## Common Mistakes

1. **Treating `sleep` as a way to "pause" a shared object.** It does not release the monitor, so other threads stay `BLOCKED` while the sleeper naps. Sleep and wait are not the same verb.
2. **Reading `RUNNABLE` as "the thread is on a core."** It is not, and chasing a spin that shows `RUNNABLE` as a hang takes you down a wrong path; the thread may just be repeatedly descheduled.
3. **Confusing `BLOCKED` with `WAITING` in a dump.** The two need different fixes: release a lock versus send a notification.

## Interview Perspective

Interviewers ask the states to see if you can translate a problem into a thread's position. Weak: listing the six names. Strong: "a thread in WAITING is parked for a signal and the fix is the notifier; a thread in BLOCKED is queued on a monitor and the fix is the lock holder." They love asking what `sleep` does to a held lock, because the person who says "releases it" reveals they memorized the diagram instead of the semantics.

Follow-up: "what does a deadlock look like in a dump" and "why does RUNNABLE not tell you the thread is running."

## Knowledge Check

- A thread is shown in `BLOCKED` on an object monitor. Name the concrete condition and the concrete fix.
- Two threads call `wait()` on the same object. You call `notify()` once. What is true about the waiters after, and what changes with `notifyAll()`?
- A thread does `sleep(5000)` while holding a lock. What states do other threads take, and why is this not a timeout for them?

## Key Takeaways

- The six states are the JVM's answer to "what is this thread doing now," and each names the exact wait.
- `BLOCKED` waits on a lock, `WAITING` waits on a signal, and the fixes are different.
- `sleep` does not release the monitor, and `RUNNABLE` does not mean on a core.

## What's Next

Now that a thread has a lifecycle, the next article answers the obvious question: how do you actually create the thing. The thread class versus Runnable versus Callable is the decision every Java program makes first, and the choice changes whether your work can return a value, throw, and be scheduled by a pool.

---

This article explains the six Java thread states and the transition each one needs. It argues that reading a thread dump is a matter of asking whether the stuck thread waits for a lock, a signal, or a timer, and that the wrong guess is usually the difference between the wait and the notify.