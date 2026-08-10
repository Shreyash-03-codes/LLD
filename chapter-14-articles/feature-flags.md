# Feature Flags

## Learning Objectives

- Separate a code change from a release decision: a flag is a runtime bool that lets new behavior ship dormant and turn on without a deploy.
- Design a flag with a rollout curve, a kill switch, a truth source, and an owner, and know the difference between a flag and a configuration setting.
- Argue for deleting dead flags on a schedule, because an orphaned branch in the codebase is debt the next reader pays.

## Introduction

A feature flag is a runtime switch that decides which code path a request takes. The idea is unremarkable and enormous at once: a flag decouples "the new code is in the build" from "the new code is live." You can deploy a behavior that is off, turn it on for one percent of users, watch it, ramp to a hundred, and flip it back to zero in a single move when it misbehaves. Before flags, the only escape from a bad deploy was a rollback: slow, blunt, and usually dragging the last good build behind it. After flags, the emergency brake is a configuration value, a seconds decision instead of a deploy.

## Problem Statement

A new payment flow has lived in a branch for six weeks, and the merge is a wall of conflicts. The team ships it, and an edge case breaks checkout in production. The options now are to revert the whole merge, losing the good changes that rode along, or carve a hotfix, and either takes minutes to tens of minutes while real users trip on the broken path. The same story with flags: the payment flow sits behind `checkout.v2`, live for a 5 percent cohort, the dashboard turns red, and one configuration flip darkens the whole feature in seconds. The flag did not make the feature better. It changed how quickly a bad feature could be withdrawn.

## Core Concept

### A flag is a decision point with an owner

A flag checks the runtime and forks behavior:

```java
if (featureFlags.isOn("checkout.v2")) {
    return v2CheckoutFlow(request);
}
return v1CheckoutFlow(request);
```

That branch is not decoration. It is a decision point with a fork, and it needs an owner, a short description, a rollout plan, and an expiry date, and that metadata lives with the flag, not in one engineer's head. The provider at that runtime can be a property file, a config service, or a dedicated flag platform; the code shape stays identical regardless. Someone has to be accountable for the branch, and the flag record is where that accountability is written down.

### What kinds of flags exist

Flags are not one thing, and treating them as one thing is where the design literature starts:

- **Release flags** gate a new behavior until it is proven, shipped off, ramped to a percent, killed at the first sign.
- **Ops flags (kill switches)**: disable an expensive code path, a cache, or a dependency call during an incident. These often stay around forever as an emergency brake.
- **Experiment flags**: split the population for a comparison, and they need stable segmentation, because an A-B result is only as good as its cohort is stable.
- **Permission flags**: feature gates per tenant or plan, enterprise features available to paying customers. These are configuration pretending to be flags, and they run on a different lifecycle from release ramps.

Mixing the lifecycles is the failure: a temporary release flag that stays on for a year and a permanent tenant flag that gets expired on a schedule will, between them, delete something that should have stayed.

### Rollout and the kill switch

A rollout curve is a ramp: 1 percent of users, then 5, then 25, then 100, with pauses for the dashboard to catch up. Two mechanics make the percentage real. The ramp needs to be sticky: a user assigned to the 1% bucket stays in it for every request and across instances, or 1% is not a cohort, it is a random coin flip per request. And the ramp needs to respect the environment and the rollout of the deploy itself: a region that just got the new build should not jump to 100% the instant it comes up.

```java
boolean isInCohort(String userId, String flag) {
    int bucket = Hashing.murmur3_32(userId).asInt() % 100;
    return bucket < featureFlags.getPercent(flag);
}
```

The hash is what gives stickiness without shared state; the same user maps to the same bucket on every instance, so the 1% is the same 1% for every request. That is the difference between measuring a rollout and annoying a random slice of users.

The kill switch is the whole promise of the flag. The first red dot on the dashboard, the flag goes to zero and the behavior darkens within the provider refresh interval, system-wide, with no deploy. The flag does not prevent the bug; it caps the blast radius and turns the recovery from a deploy into a knob. That is why a good kill switch is a dedicated low flag tied to nothing else: when the incident is live, you do not want "99% off but 1% still bleeding," you want one knob that is entirely dark or entirely on.

### Dead flags are deleted, not kept

The slow death of a flag suite is entropy compounding. A flag ramped to 100% and left in the code now has a branch that is always taken, two paths that do the same thing, and a config entry nobody manages. Every future edit to that class has to read "if flag.isOn(...)" and think about an unknown. The discipline: a flag gets an expiry date the day it is created, and the deletion task is on the calendar like any other cleanup. "We should remove that flag someday" is a debt sentence; "this flag expires on the 22nd and the issue is filed" is a flag that will actually be deleted.

### The default when the provider is down

The flag service is a dependency, and dependencies fail. Rule: a release flag defaults to OFF when the provider is unreachable, and an ops kill switch defaults to its fail-safe state, which for a kill switch like "disable the dependent call" is ON. The wrong default in either direction is one of two failure modes:

- Default OFF on a release: the feature comes up dark even as the intent flips, and users never see a feature the team believed was live.
- Default ON on a release: the feature comes up fully live at the moment you most need the kill switch to actually kill.

Both defaults are real choices. The bug is the default that was never chosen: the one that happens to be exactly the wrong direction at the exact load when the provider is down.

## Real Production Usage

The Java-side constellation is small. Spring Boot reads `@ConditionalOnProperty` for the simplest cases, and the `Environment` abstraction pipes a property file or a config server. Beyond a literal boolean, teams use config services (Spring Cloud Config, or a vendor feature-flag provider like LaunchDarkly, Unleash, or GrowthBook, which all expose a Java client and centrally host the percent, the targeting, and the audit). The richer infrastructure matters when the flag becomes real: targeting rules per user attribute, percentage ramps, audit logs of who flipped what, and an SDK hot path that caches. The test that matters at the deliverable is a flag that is worthless if a flip takes a minute to reach the instances, because the kill switch is only as fast as the propagation.

The flags that survive into a mature service are a few release ramps, a few kill switches, and the per-tenant flags, and everything else was deleted. Teams with healthy flag hygiene keep a flags page because the list is itself a health metric: a page with thirty untouched flags is a page nobody reads.

## Common Mistakes

1. **Leaving a flag pinned at 100% and calling it done.** A flag ramped so every user gets the new behavior is not "released with a rollback handle," it is a permanent junction in the code that always takes one branch. Once the ramp hits 100 and the dashboard is green, the deletion removes the old path and the condition with it, expiry first.
2. **A default that trusts the flag service.** The most dangerous flags default ON when the provider is unreachable, so the failure of the config system is the moment the whole feature comes 100% open. The fail-safe defaults are chosen per flag, and each has to be decided, and the dangerous assumption is the one you never made.
3. **Rollout that is not sticky.** Ramping to 5% but assigning a fresh random each request means the user hops in and out of the feature on the same session, the experiment bucketing is garbage, and the "5%" is a rule nobody can reproduce.

## Interview Perspective

The interviewer uses the flag question to check whether "we keep it behind a flag" is a sentence with content. Weak: "we put the boolean in the environment." Strong: "release flag, ramped by a sticky per-user cohort, a dedicated kill switch, a default that fails safe, and an expiry date on the flag so it becomes a deletion task." The follow-ups they probe are "when do you delete a flag" and "what does sticky mean," which separate someone who has run a rollout from someone who knows the word.

## Knowledge Check

1. The flag service is unreachable for five minutes. The release flag's default is ON. What does traffic actually see while the provider is down, and why was that default a design choice?
2. A PR adds six flags, all default OFF, no owner, no expiry. A month later, which of these flags is the most dangerous and why?
3. The experiment concludes "keep the new path." What is actually required to remove the code and the flag, and what is the danger of a flag still at 100% in six months?

## Key Takeaways

- A flag decouples shipping from releasing: the behavior ships dark, ramps by a sticky cohort, and is withdrawn with a config flip, not a deploy.
- Kill switch defaults are designed, not inherited; a release flags can fail to OFF, an ops switch to its safe, and any doubt of the flag service is the failure that is defined.
- Every flag starts with an expiry and an owner, and a flag at 100% is debt, not optional leftover.

## What's Next

This chapter has built the pieces of reliability one at a time, logging, audit, metrics, health, retries, breakers, rate limits, flags. The final article is where the pieces stop being ingredients: designing for observability puts it all together, how logs, metrics, traces, health, and flags are wired into one story that answers a production question, and how the wiring is itself a design, not an accumulation.

---

This article explains feature flags as a runtime gate that separates shipping from enabling a feature. It argues that a release flag should fail safe when its provider is down, and that a flag locked at 100% is dead code waiting to be deleted.
