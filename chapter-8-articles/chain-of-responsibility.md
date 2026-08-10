# Chain of Responsibility

## Learning Objectives

- Build a chain of handlers where a request passes down an ordered list until a handler claims it, and each handler can act, pass on, or stop.
- Define the two guarantees that keep a chain honest: every handler has a defined successor, and the chain has a default answer when nothing claims the request.
- Explain the servlet filter chain and Spring Security's filter chain as the pattern in daily production use.

## Introduction

Chain of Responsibility gives a request a sequence of potential handlers. The request enters the head of the chain and each handler decides: handle it and stop, handle it and pass the result on, or pass it untouched to the next handler. The requester never knows which handler took the request. The requester only knows to send it into the chain head.

The pattern's gift is decoupling the sender from the handler. A sender does not hardcode "the CSRF filter, then the auth filter." It hands the request to the front of the chain, and each filter decides its own level of involvement. Adding a handler does not touch the sender.

## Problem Statement

The failure is a dispatch growing longer than any one object should own. A request travels through a validation pipeline, and the pipeline is a method with a stack of checks:

```java
public Response handle(HttpRequest request) {
    if (!auth.isAuthenticated(request)) {
        return auth.deny(request);
    }
    if (!csrf.isValid(request)) {
        return csrf.reject(request);
    }
    if (!rateLimiter.allow(request)) {
        return rateLimiter.throttle(request);
    }
    // ... six more checks, each a new branch
    return controller.dispatch(request);
}
```

Ten checks, twelve ifs, one method. Every new concern means editing this method, and the method's tests must cover every combination of checks in every order. And order matters, authorization must precede the rate limit, so a reorder silently changes the security posture. The method has become both the list of concerns and the code that runs them, and it will keep growing until it is a checklist nobody can read.

The deeper failure is that the decision, "handle this or pass it on," is written directly into the request flow. Each check wants to be independent, the auth check should not need to know the rate limiter exists, but in this method they are all one block, and none of them can be added, removed, or reordered without editing the whole.

## Core Concept

Chain of Responsibility gives each check its own class and links them into a chain. One interface names the handler:

```java
public interface Filter {
    void doFilter(HttpRequest request, FilterChain chain);
}
```

Each handler holds the reference to the next step, and decides whether to handle, stop, or pass:

```java
public class AuthFilter implements Filter {
    private final AuthenticationService auth;

    public AuthFilter(AuthenticationService auth) {
        this.auth = auth;
    }

    @Override
    public void doFilter(HttpRequest request, FilterChain chain) {
        if (!auth.isAuthenticated(request)) {
            request.setRejected("unauthenticated");
            return; // stop here
        }
        chain.doFilter(request); // pass to the next handler
    }
}
```

The chain is built once, ordered, at the wiring point:

```java
Filter auth = new AuthFilter(new DatabaseAuth());
Filter csrf = new CsrfFilter(secret);
Filter rate = new RateLimitFilter(store);

auth.setNext(csrf);
csrf.setNext(rate);
```

Or, with a chain builder, because the most irritating part of the pattern is remembering to link the last one:

```java
FilterChain chain = FilterChain.of(auth, csrf, rate);
```

The recursion is the mechanics. `doFilter` either handles, stops, or calls the next `doFilter`. The chain ends either when a handler claims the request or when it runs past the last handler, at which point the terminal act, the actual dispatch, happens. The sender calls the head once, and the decision, "who does what in what order," lives entirely in the chain's construction.

Diagram: request passing through handlers

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 660" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="150" y="30" width="200" height="60" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="250" y="54" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Request</text>
  <text x="250" y="76" text-anchor="middle" font-size="12" fill="#1a2733">enters chain</text>

  <line x1="250" y1="90" x2="250" y2="148" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="150" y="150" width="200" height="70" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="250" y="176" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">AuthFilter</text>
  <text x="250" y="202" text-anchor="middle" font-size="12" fill="#1a2733">handle or pass</text>

  <line x1="250" y1="220" x2="250" y2="278" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="150" y="280" width="200" height="70" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="250" y="306" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">CsrfFilter</text>
  <text x="250" y="332" text-anchor="middle" font-size="12" fill="#1a2733">handle or pass</text>

  <line x1="250" y1="350" x2="250" y2="408" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="150" y="410" width="200" height="70" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="250" y="436" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">RateLimitFilter</text>
  <text x="250" y="462" text-anchor="middle" font-size="12" fill="#1a2733">handle or pass</text>

  <line x1="250" y1="480" x2="250" y2="538" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="150" y="540" width="200" height="70" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="250" y="566" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Terminal</text>
  <text x="250" y="592" text-anchor="middle" font-size="12" fill="#1a2733">dispatch request</text>
</svg>
```

The request enters `AuthFilter`, and each handler either stops or passes vertically down to the next. The chain terminates at a handler with no successor, the terminal, so the "nobody handled it" case is a real destination, not a silent drop. That terminal is the piece everyone forgets.

### The rules that keep it honest

A chain rots in predictable ways, and three rules hold it together.

The handler must have a successor or be terminal. If a chain of three handlers can silently end because handler two forgot to set a next, requests vanish. The chain builder that rejects an unlinked tail, or a terminal that is always present, makes the drop impossible.

The ordering must be load-bearing and deliberate. The chain is only correct in the order it was built. In the servlet world this is famously strict: authentication must precede authorization, CSRF must run before the filter can trust the request. Reordering the chain is a security change, not a cosmetic shuffle.

The handler must do one thing. A handler that both validates and transforms and logs is three handlers in one coat. The whole point of the pattern is that a new concern is a new handler added to the list. A handler that accretes concerns rebuilds the monolithic method the pattern exists to break apart.

## Real Production Usage

The servlet filter chain is the canonical production chain, and it was named in a previous chapter. In a servlet container the `Filter` classes form a chain: each `doFilter` either handles the request or calls `chain.doFilter(request, response)` to pass it toward the servlet. Your web framework is a consumer of that chain. Spring Security builds its filter chain on top of it, where each `SecurityFilterChain` is an ordered list of filters and the ordering is enforced and documented because the pattern's correctness depends on it.

`java.util.logging` uses the chain form. A `Logger` records a message and passes it to its parent logger, all the way up the hierarchy, and any level can filter or stop the propagation. The `Handler` delegation is the pattern. Spring MVC's `HandlerInterceptor` list is a chain: each interceptor decides before and after a handler method, passes on, or short circuits. Netty's event pipeline is a chain too, each `ChannelHandler` does its work and passes the message on, or drops it. When you see a framework talk about "the chain," "the pipeline," or "an ordered list of steps," you are looking at this pattern. One request, many possible handlers, ordered. Middleware, all of middleware, is built on this chain.

## Common Mistakes

**A chain that falls off the end.** If the last handler passes to nothing and the default action lives in no handler, valid requests get dropped silently. A terminal handler is not optional.

**Brittle ordering.** The chain's correctness lives in its order. If a handler passes when it should stop, the request reaches the wrong terminal and the state it was supposed to reject flows onward. Ordering is part of the spec, and a chain built in the wrong order is a broken chain, not a stylistic choice.

**Putting coordination in the handlers.** Each handler should only know its own concern plus the chain call. The moment a handler starts inspecting others' decisions or reordering work, the chain has become the monolith again with extra classes.

## Interview Perspective

Chain of Responsibility is the pattern interviewers use to check ordering thinking. A strong answer leads with the terminal and the ordering, the two things that make a chain correct, and can say "the servlet filter chain, and Spring Security on top of it." A weak answer draws three boxes and assumes the chain handles itself. The pattern is easy to misread as "a list of steps," so the candidate who explains why a bare list is not enough, because a list has no terminal and no guarantee that anything claims the request, is the one who has actually built one.

The follow up that sorts people: "The same request, and each handler gets a chance. Why would you prefer CoR over Observer?" The strong answer: CoR is ordered and short-circuits, exactly one handler acts, and the sender does not know who; Observer notifies everyone, unordered, and never short-circuits. Choosing Observer when you need a single ordered actor is the wrong pattern for the problem.

Common follow-ups:

- "Every handler runs even if one rejects. Where does that break the pattern?"
- "Why is the order of the security filter chain not cosmetic?"

## Knowledge Check

1. Draw the call chain as `AuthFilter`, `CsrfFilter`, `RateLimitFilter`, `Terminal` for an authenticated request, and mark where each one invokes `chain.doFilter`.
2. Change the order so `RateLimitFilter` runs before `AuthFilter`. What breaks, and what does the order tell you about the pattern's assumptions?
3. `java.util.logging.Logger` passes a record up to its parent. Where does this chain end, and what acts as the terminal?

## Key Takeaways

- CoR passes a request down an ordered list of handlers, each choosing to handle, stop, or pass, and the sender never knows who took it.
- The terminal handler is not optional; unhandled is a real outcome that must have a destination.
- Ordering is part of the correctness. Reordering a chain is a semantic change, not a shuffle.
- A handler does exactly one logical test, and a handler that accretes concerns rebuilds the monolith.
- Servlet filters and Spring Security are CoR in production, and their ordering is enforced precisely because it matters.

## What's Next

The next article is Template Method, which solves a different part of the same problem. Where CoR orders separate handlers, Template Method fixes the skeleton of one algorithm in a base class and lets subclasses fill in only the steps that vary. We will cover the fixed-versus-variable split, the `abstract` method seam, and the one place this pattern is a real trap instead of a design.

---

This article explains the Chain of Responsibility pattern as passing a request down an ordered list of handlers until one claims it. It argues that the terminal handler and the deliberate ordering are what make a chain correct.