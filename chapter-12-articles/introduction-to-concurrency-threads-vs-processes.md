# Introduction to Concurrency: Threads vs Processes

## Learning Objectives

- Distinguish a process from a thread by their memory isolation, not by which one is "lighter."
- Explain that threads in one Java process share the JVM heap, and state the two consequences: they can cooperate and they can corrupt each other.
- Say why a thread exists at all, and be able to argue that "it blocks on I/O" is the reason, not the failure.

## Introduction

Concurrency is a program doing more than one thing at a time, and it runs on two units: the process and the thread. Both let work run in parallel. They differ in one property that colors everything in this chapter: what each isolates. A process has its own address space. A thread shares the address space of the process that owns it. That single fact is the root of every hard problem you will meet in this book, because it is the reason concurrent code can be simultaneously fast and catastrophically wrong.

## Problem Statement

Two separate programs each count visits into their own variable, run side by side, never interfere. Now collapse both counters into one shared variable across two threads. The increment is not atomic. Each step, read the value, add one, write it back, can interleave between the two threads, and the final value can be less than the number of clicks you recorded. That is a lost update. The isolation that protected you, the process, is gone, and nothing in the code says why. This chapter is about the tools you build to win back some of that boundary when sharing is the whole point of the design.

## Core Concept

The difference is owning memory.

A process is an execution unit with its own address space. Its code, its data, its heap, its stack all belong to it alone. If process A writes a byte, process B never sees it unless the two explicitly share a pipe, a socket, or a named segment of memory. That isolation is a property of the OS, and it is the reason a crash in one process usually leaves its neighbors intact.

A thread is an execution unit inside a process. All the threads of one process share the code, the data, and the heap of that process, and each thread keeps its own stack and its own run-time bookkeeping. So two threads of one program can both hold a reference to the same object and call it. That is the cooperation. It is also, without care, the way they tear each other apart.

Diagram: a process isolates all of code, data, heap, and stack, while the threads of one process share code, data, and heap and keep a stack each.

<svg width="920" height="360" viewBox="0 0 920 360" xmlns="http://www.w3.org/2000/svg">
  <rect x="40" y="50" width="230" height="230" rx="10" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="155" y="72" text-anchor="middle" font-family="Arial" font-size="14" font-weight="bold" fill="#222">Process A</text>

  <rect x="56" y="88" width="188" height="36" rx="6" fill="#ffffff" stroke="#555" stroke-width="1.5"/>
  <text x="150" y="112" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Code</text>
  <rect x="56" y="132" width="188" height="36" rx="6" fill="#ffffff" stroke="#555" stroke-width="1.5"/>
  <text x="150" y="156" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Data</text>
  <rect x="56" y="176" width="188" height="36" rx="6" fill="#ffffff" stroke="#555" stroke-width="1.5"/>
  <text x="150" y="200" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Heap</text>
  <rect x="56" y="220" width="188" height="36" rx="6" fill="#ffffff" stroke="#555" stroke-width="1.5"/>
  <text x="150" y="244" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Stack</text>

  <rect x="310" y="50" width="230" height="230" rx="10" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="425" y="72" text-anchor="middle" font-family="Arial" font-size="14" font-weight="bold" fill="#222">Process B</text>

  <rect x="326" y="88" width="188" height="36" rx="6" fill="#ffffff" stroke="#555" stroke-width="1.5"/>
  <text x="420" y="112" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Code</text>
  <rect x="326" y="132" width="188" height="36" rx="6" fill="#ffffff" stroke="#555" stroke-width="1.5"/>
  <text x="420" y="156" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Data</text>
  <rect x="326" y="176" width="188" height="36" rx="6" fill="#ffffff" stroke="#555" stroke-width="1.5"/>
  <text x="420" y="200" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Heap</text>
  <rect x="326" y="220" width="188" height="36" rx="6" fill="#ffffff" stroke="#555" stroke-width="1.5"/>
  <text x="420" y="244" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Stack</text>

  <rect x="600" y="50" width="300" height="230" rx="10" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="750" y="72" text-anchor="middle" font-family="Arial" font-size="14" font-weight="bold" fill="#222">One process, two threads</text>

  <rect x="616" y="88" width="268" height="36" rx="6" fill="#ffffff" stroke="#8a6d00" stroke-width="1.5"/>
  <text x="750" y="112" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Code (shared)</text>
  <rect x="616" y="132" width="268" height="36" rx="6" fill="#ffffff" stroke="#8a6d00" stroke-width="1.5"/>
  <text x="750" y="156" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Data (shared)</text>
  <rect x="616" y="176" width="268" height="36" rx="6" fill="#ffffff" stroke="#8a6d00" stroke-width="1.5"/>
  <text x="750" y="200" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Heap (shared)</text>

  <rect x="616" y="224" width="126" height="40" rx="6" fill="#ffffff" stroke="#8a6d00" stroke-width="1.5"/>
  <text x="679" y="248" text-anchor="middle" font-family="Arial" font-size="12" fill="#222">Thread 1 stack</text>
  <rect x="758" y="224" width="126" height="40" rx="6" fill="#ffffff" stroke="#8a6d00" stroke-width="1.5"/>
  <text x="821" y="248" text-anchor="middle" font-family="Arial" font-size="12" fill="#222">Thread 2 stack</text>
</svg>

The everyday English that comes out of this diagram: in Java, the threads of one JVM share the heap. Two threads writing to one object can corrupt each other's work. Two different JVMs can never see each other's objects at all. So isolation is what the process gives you for free, and speed plus correctness taxes are what threads give you in exchange.

### The process protects, the thread needs to share

Pack this into a line worth saying out loud: cross-process you get isolation for free, and cross-thread you get sharing, plus the responsibility to keep that sharing correct. The JVM is the boundary. If you want to avoid all races, the most radical answer is to put the state in separate processes and pass events between them, which is what services and message queues end up doing. That is the isolation design pattern. When you choose threads, you choose shared memory, and the rest of this chapter assumes you did.

### The actual reason threads exist

The real reason a thread exists is waiting. When a thread blocks on a database call or a socket, the CPU is idle for that thread, and another thread can hop on the core and make progress. Without that, one slow disk call would stall the entire program. The whole point of threads is overlapping I/O with compute. The "it blocks on I/O" that beginners treat as a problem is the entire reason the machine makes sense. The rest of this chapter is the machinery that makes that overlap safe.

## Real Production Usage

Every Spring web application is running on threads. The container keeps a pool of request threads, one per in-flight request, each blocking on its database calls while the rest keep handling new arrivals. The platform threads, the classic `java.lang.Thread`, map one to one onto OS threads that the operating system schedules. Java 21 added virtual threads, which lets a single JVM hold tens of thousands of them with a tiny footprint each, but they all still share one heap, so the shared memory reasoning of this article does not change. What changes is how many threads you can cheaply wait with.

## Common Mistakes

1. **Thinking threads are fast so they must be a lot.** Threads are cheap to create, but the cost is coordination. The mistake is adding threads to a CPU-bound loop, where they serialize on the single core anyway and add worry for nothing.
2. **Treating a process and a thread as the same unit.** Then the sharing in the process is not questioned, because the caller assumed it did not exist.
3. **Reaching for the process when the state has to be shared.** You get isolation but now you have to build a whole communication path you are not ready for.

## Interview Perspective

For the senior LLD about "process vs thread", they want memory isolation and why you might say one is right. The weak answer lists speed as both. The strong one says process isolates address space for free, thread shares the heap and that sharing is a correctness tax, and you pick by whether the state must be shared. They like to hear the phrase, "threads share the heap, processes do not."

Follow-ups to expect: "when is a process worth the weight" and "does Java 21 virtual thread remove the shared memory question." The second is the trap: virtual threads change the cost of waiting, not the fact of sharing.

## Knowledge Check

- Two threads call `++visits` on one field. Explain, in terms of isolation, why the count can be low and what unit of sharing is at fault.
- Which memory regions do the threads of one process share and which are per thread?
- A library ships one process per job. Why, per this article, is that isolation a deliberate cost rather than an accident?

## Key Takeaways

- A process owns its address space; threads of a process share the heap and keep their own stack.
- Shared state is why threads race; separate processes give isolation for free.
- Threads exist because blocking I/O lets the core be used by another thread.

## What's Next

The thread is the unit this chapter centers on, and the next article opens the shape it moves through. The thread lifecycle walks you through the states every thread passes and gives you a vocabulary for what a thread does while it waits, states the rest of the chapter will reuse.

---

This article explains the line between a process and a thread as a question of memory ownership. It argues that threads exist to overlap I/O with compute, and that the heap sharing that makes them cheap is the same sharing that turns a simple counter wrong.