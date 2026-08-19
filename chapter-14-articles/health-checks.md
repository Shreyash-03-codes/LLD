# Health Checks

## Learning Objectives

- Separate liveness from readiness and know which one restarts a pod and which one takes it out of the load balancer.
- Design a health check that verifies real dependencies without making a database outage into a distributed crash.
- Reason about health under a rolling restart without either dropping traffic or routing to an instance that cannot serve.

## Introduction

Health checks are the coordination between a service instance and the infrastructure that routes to it. A load balancer and an orchestrator both need an answer to a tiny question: "can I send this test-traffic to this instance right now?" The endpoint exists to answer that, and it must be cheap, precise, and single-purpose. The most common failure is an endpoint that answers "yes" when the instance is broken, or "no" when it is fine, and either way the orchestrator trusts it. Getting health checks wrong converts a small failure into a cascade.

## Problem Statement

A deploy rolls out instance by instance, and the orchestrator waits for each new instance's health check to pass before moving on. The health endpoint checks the database on every call. Mid-deploy, the database driver hits a slow moment, three probes time out, the orchestrator declares the new instance unhealthy, kills it, rolls the old one back, and the deploy restarts. Meanwhile the load balancer, watching the same endpoint, starts draining the whole fleet because every instance's probe now competes for a database connection. One dependency hiccup turned into a fleet-wide restart. The health check, built to protect the service, was itself the amplifier.

## Core Concept

Two answers, two verbs.

**Liveness** answers "is the process alive." It measures the bare minimum: the JVM is up, the main loop is running, the process has not deadlocked. A liveness failure is usually permanent and the correct action is to restart the thing. Kubernetes probes it with `/health/live` and restarts the container on failure.

**Readiness** answers "can I take traffic right now." It measures whether the instance can currently serve a request: it can connect to the database, it has finished warmup, its dependency quota is not exhausted. A readiness failure is usually temporary, the correct action is to stop routing to it, not to kill it, and Kubernetes uses `/health/ready` for this and leaves the container running.

The two confusions that cause most incidents:

1. One endpoint for both. A single `/health` that the orchestrator uses for both kill and routing. A temporary readiness dip, like a DB blip, then kills pods instead of draining them.
2. Readiness checking dependencies that are not needed to serve. If the service uses the database but not for the health path itself, failing readiness on a DB blip takes it down with the database.

The rule: liveness must be independent and never depend on anything outside the process. If liveness checks the database, then when the database wedges, every instance fails liveness, and the orchestrator restarts everything in parallel. That is the distributed crash. Liveness checks the process; readiness checks the things the process needs to serve.

```java
@Component
public class ReadinessProbe {
    private final DataSource dataSource;
    private final AtomicBoolean warm = new AtomicBoolean();

    @Scheduled(initialDelay = 15_000, fixedDelay = 15_000)
    public void warmUp() {
        warm.set(true);
    }

    @GetMapping("/health/ready")
    public String ready() {
        if (!warm.get()) {
            return "warming";        // take self out of rotation until this clears
        }
        return "ready";
    }
}
```

The shape of a healthy pairing:

Diagram: liveness keeps the pod alive on process failure, readiness moves traffic away on dependency failure.

<svg width="860" height="330" viewBox="0 0 860 330" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="hc" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#333"/>
    </marker>
  </defs>

  <rect x="40" y="120" width="150" height="70" rx="6" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="115" y="150" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Instance</text>
  <text x="115" y="170" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">one pod</text>

  <rect x="280" y="50" width="200" height="55" rx="6" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="380" y="75" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Liveness probe</text>
  <text x="380" y="94" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">process alive?</text>

  <rect x="280" y="195" width="200" height="55" rx="6" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="380" y="220" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Readiness probe</text>
  <text x="380" y="239" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">can take traffic?</text>

  <path d="M 190 140 L 272 82" stroke="#333" stroke-width="2" fill="none" marker-end="url(#hc)"/>
  <path d="M 190 170 L 272 217" stroke="#333" stroke-width="2" fill="none" marker-end="url(#hc)"/>

  <rect x="565" y="45" width="240" height="65" rx="6" fill="#fdeeee" stroke="#962828" stroke-width="2"/>
  <text x="685" y="70" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Restart pod</text>
  <text x="685" y="89" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">failure is permanent</text>

  <rect x="565" y="190" width="240" height="65" rx="6" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="685" y="215" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Drain traffic</text>
  <text x="685" y="234" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">instance keeps running</text>

  <path d="M 480 82 L 562 88" stroke="#962828" stroke-width="2" fill="none" marker-end="url(#hc)"/>
  <path d="M 480 217 L 562 213" stroke="#2f6f3e" stroke-width="2" fill="none" marker-end="url(#hc)"/>
</svg>

Notice the asymmetry. A liveness failure kills the pod because a dead process cannot be drained. A readiness failure only stops the routing; the process keeps running and can recover. The pairing is what lets a dependency blip pass through the fleet without a restart.

### What the probe should and should not contain

A readiness probe that opens a new database connection on every call is a load generator, not a check. The database will count thousands of probe connections a minute, and at the moment the database struggles the probes are the extra load tipping it over. Teams build dependency checks fine-grained; the mature shape is a cached or bounded dependency probe that checks the dependencies the request path actually uses, and it runs slowly, once every ten seconds, not per request.

The more subtle rule: a readiness failure should mean "take me out of rotation," not "restart me." So the probe must be able to recover without restarting. A database blip then drains the instance, the database recovers, the probe starts passing and the orchestrator sends traffic again. That is the correct cycle. Testing it: stop the database, observe readiness flips to false and liveness stays true. That's the proof your health checks are wired correctly.

A signal to label: the state of the two probes is itself a metric. "readiness false" vs "liveness failed" distinguishes, and you alert on readiness changes and not liveness.

## Real Production Usage

Spring Boot is the standard: the `/actuator/health` group can expose liveness (`/actuator/health/liveness`) and readiness (`/actuator/health/readiness`), with a separate `livenessState` that keeps the process check independent. Kubernetes configures `livenessProbe` and `readinessProbe` hitting those paths, with `initialDelaySeconds` covering warmup. The mature setup keeps warmup off the readiness gate with an initial delay, rate-limits the DB check or removes it, and uses a probe timeout smaller than the orchestrator's. The two probes have different semantics: a liveness timeout means the container is wedged and should restart; a readiness timeout means the instance is busy and should be drained, not killed.

## Common Mistakes

1. **One health endpoint doing double duty.** A single `/health` used for both liveness and readiness: a database blip kills pods instead of draining them. Split them.
2. **A liveness probe that depends on the world.** When the dependency fails, all instances fail liveness at once and the orchestrator restarts them in parallel, a distributed crash.
3. **Health path itself becomes the bottleneck.** A probe that runs a DB query on every call, or a probe endpoint that passes through the full filter chain with auth and rate limiting, turns the health check into load on the very thing it measures.

## Interview Perspective

The question is "liveness and readiness, what's the difference, and how do you wire them." The strong answer: "liveness is about the process, a failure means restart, and it must not depend on anything external; readiness is about serving, a failure means pull traffic, and it can depend on dependencies but must be rate-limited and recoverable." The weak answer mixes the two, calling readiness a restart condition. A follow-up that equals the level: "your readiness probe queries the DB, what happens during a DB blip" wants the wire draining, not restarting.

## Knowledge Check

1. Your liveness probe performs the same DB query as readiness. Pick a moment when both endpoints have an outage and describe the difference in what the orchestrator does, and the cascading state after.
2. A new instance's startup takes 60 seconds of warmup. What happens if there's no initial delay on readiness and no warm flag, and describe the two ways this is a failing move.
3. Where does the health check's check itself belong in the "saturation" story: explain the difference between a readiness check, a metric that looks distressed and rolling.

## Key Takeaways

- Liveness = process is alive, failing it is a restart; readiness = can serve, failing it drains, not restarts.
- Liveness must not depend on resources; readiness can but uses a bounded, rate-limited probe that can recover.
- The two-failure distinction is the reason probes and a rolling deploy live or die.

## What's Next

Health checks answer a binary question about one instance. The next article crosses instance boundaries and adds time: retry mechanisms and backoff strategies handle a call that failed on a temporary blip, and how retrying the right way does what the health check cannot, surviving a dependency that is coming back not yet up.

---

This article explains the two health probes, liveness and readiness, and why conflating them turns a dependency blip into a fleet restart. It argues that liveness must never depend on the outside world, or the probe becomes the amplifier it exists to prevent.
