# Real-World Usage: How Spring Security Uses Filters and Proxies

## Learning Objectives

- Trace a request from the servlet container through Spring Security's filter chain to a controller, naming the structural pattern at each hop.
- Explain `DelegatingFilterProxy` and `FilterChainProxy` as proxies, and the security filters as decorators of the request and response.
- Connect Spring's AOP proxies, JDK dynamic and CGLIB, to the Proxy pattern, and predict the failure modes that follow from them.

## Introduction

This is the chapter's payoff article. The seven structural patterns have been drawn in clean diagrams, and this is where you meet them in a framework you will actually run, Spring Security. It is a structural pattern showroom, and it does not name any of the patterns. `DelegatingFilterProxy`, `FilterChainProxy`, `SecurityContextHolderAwareRequestWrapper`, they are all pattern names wearing class names.

The skill this chapter has been building, recognizing the shape and naming the intent, is exactly what you need to read a request through this framework without getting lost in the acronyms.

## Problem Statement

The failure is the one every Spring developer hits eventually. You add a security filter to the chain and it does nothing. Or you annotate a method `@PreAuthorize` and it is bypassed. Or you log the transaction inside a service method and the outer transaction never rolls back. In every case the code looks right, the annotation is right, and the behavior is wrong, because there is a layer of proxies and filters between your code and what you think you wrote.

You cannot debug what you cannot see. Until you know that your bean is wrapped in a proxy, that the request is being decorated as it passes through filters, and that the filter chain has an order that matters, Spring Security looks like magic with occasional bugs. The patterns are the map. This article draws it.

## Core Concept

### The servlet filter chain

Everything starts with the servlet container, Tomcat or Jetty. It receives an HTTP request and passes it through a chain of `javax.servlet.Filter`s before it ever reaches a servlet. Each filter can inspect the request, wrap it, and pass it on:

```java
public class TimingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request,
                         ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        long start = System.nanoTime();
        chain.doFilter(request, response);
        long elapsed = System.nanoTime() - start;
        log.info("handled in {} ns", elapsed);
    }
}
```

`chain.doFilter(...)` passes the request to the next filter, and when the last one calls it, the request reaches the servlet. The return path unwinds in reverse. This is Chain of Responsibility, a behavioral pattern that the next chapter will cover properly, but structurally it is the container that holds the chain.

### DelegatingFilterProxy: a proxy across two worlds

Here is the first pattern in disguise. Spring Security's filters are Spring beans, they need dependency injection, a Spring context, and lifecycle management. The servlet container knows nothing about Spring beans. Something has to bridge the two, and that something is `DelegatingFilterProxy`, a servlet filter that the container can instantiate, whose only job is to look up a Spring bean by name and delegate to it:

```java
public class DelegatingFilterProxy implements Filter {
    @Override
    public void doFilter(ServletRequest request,
                         ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        Filter delegate = findBean("springSecurityFilterChain", Filter.class);
        delegate.doFilter(request, response, chain);
    }
}
```

This is a Proxy in the purest sense. It preserves the `Filter` interface, it stands in for another `Filter` it may not have loaded yet, and it controls when and how the real object gets involved. The client, the servlet container, cannot tell it is talking to a proxy, and the proxy handles the lookup lazily because the Spring context is bootstrapped after the container starts. Deferred, protected, hidden: all three proxy jobs in one class.

### FilterChainProxy: the chain that owns the chain

`DelegatingFilterProxy` looks up a bean named `springSecurityFilterChain`, and that bean is an instance of `FilterChainProxy`. Another proxy, this one is a Spring bean that appears to the container as a single filter and internally dispatches to a list of `SecurityFilterChain`s, one per configuration. Each `SecurityFilterChain` is a list of security filters in order: `CsrfFilter`, `UsernamePasswordAuthenticationFilter`, `AuthorizationFilter`, and the rest.

The order is load-bearing. CSRF protection must run before authentication can trust the request, authentication must run before authorization can check the principal. Spring gives you the ordering rules, but the reason they exist is structural: these filters are a chain, and a chain is only as correct as its order.

### The filters as decorators

Individual security filters are Decorators of the request and response. `ContentCachingRequestWrapper` wraps the request to buffer its body so it can be read twice, an application reads it once and the framework logs it after. `SecurityContextHolderAwareRequestWrapper` wraps the request to add security convenience methods like `isUserInRole`. Both preserve the `HttpServletRequest` interface, hold the wrapped request, add behavior, and delegate the rest. That is the Decorator pattern, and the servlet wrapper classes exist precisely so this wrapping is legal, because wrapping a concrete request class would break everything that casts to `HttpServletRequest`.

### The AOP proxies

Method security and transactions run on the third structural pattern: the AOP proxy. When Spring needs to apply an aspect to a bean, it does not modify your class. It creates a proxy: a JDK dynamic proxy if the bean implements an interface, a CGLIB subclass otherwise. Your `@PreAuthorize("hasRole('ADMIN')")` method is wrapped, and the proxy checks authorization before calling the real method:

```java
public class MethodSecurityInterceptor {
    public Object invoke(MethodInvocation invocation) throws Throwable {
        checkAccess(invocation);
        return invocation.proceed();
    }
}
```

This is a protection proxy at industrial scale. It also explains the classic Spring gotchas. Calling an annotated method from within the same class bypasses the proxy, because you are holding the plain object, not the injected proxy. A `final` class or a `final` method cannot be proxied by CGLIB, so the security just does not apply. Knowing the pattern means these are not mysteries, they are the proxy's known edges.

Diagram: request through the security filter chain

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 560 590" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="210" y="30" width="200" height="60" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="310" y="52" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">HTTP Request</text>
  <text x="310" y="74" text-anchor="middle" font-size="12" fill="#1a2733">enters pipeline</text>

  <rect x="190" y="140" width="240" height="70" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="310" y="162" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">DelegatingFilterProxy</text>
  <text x="310" y="186" text-anchor="middle" font-size="12" fill="#1a2733">looks up Spring bean</text>

  <rect x="190" y="260" width="240" height="70" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="310" y="282" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">FilterChainProxy</text>
  <text x="310" y="306" text-anchor="middle" font-size="12" fill="#1a2733">matches a chain</text>

  <rect x="160" y="380" width="300" height="80" fill="#e9f5ee" stroke="#2e7d4f" stroke-width="1.5"/>
  <text x="310" y="402" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">SecurityFilterChain</text>
  <text x="310" y="428" text-anchor="middle" font-size="12" fill="#1a2733">CSRF, AuthN, AuthZ filters</text>

  <rect x="210" y="500" width="200" height="60" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="310" y="522" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Servlet</text>
  <text x="310" y="548" text-anchor="middle" font-size="12" fill="#1a2733">handles request</text>

  <line x1="310" y1="90" x2="310" y2="138" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="310" y1="210" x2="310" y2="258" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="310" y1="330" x2="310" y2="378" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="310" y1="460" x2="310" y2="498" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <line x1="410" y1="530" x2="500" y2="530" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="500" y1="530" x2="500" y2="60" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4"/>
  <line x1="500" y1="60" x2="412" y2="60" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <text x="512" y="300" font-size="12" fill="#5a6b7a">response</text>
</svg>
```

The request descends through two proxies into the chain of filters, then reaches the servlet. The dashed path on the right is the response unwinding back the same way, which is why a filter's cleanup code, written after `chain.doFilter(...)`, runs on the way back out.

## Real Production Usage

This article is the real production usage section. The patterns are not abstract here: `DelegatingFilterProxy` and `FilterChainProxy` ship in `spring-web`, the filter chain order is configured in every Spring Boot security setup, `ContentCachingRequestWrapper` and `SecurityContextHolderAwareRequestWrapper` are in `spring-security-web`, and the AOP proxies behind `@PreAuthorize`, `@Transactional`, and `@Cacheable` are in every Spring application you have run. When you understand the patterns, a Spring Boot security configuration stops being XML you copy and becomes a description of which proxy wraps which object in which order.

## Common Mistakes

**Treating filter order as optional.** The chain is only as correct as its order, and reordering filters, or registering a raw servlet filter in the wrong position, silently disables a guarantee. The `@Order` on your filter is the chain's correctness, not a cosmetic hint.

**Expecting self-invocation to go through the proxy.** `this.doSomething()` inside a bean is a call to the plain object, no proxy, no transaction, no security. The moment you expect an annotation to apply to an internal call, you have forgotten the proxy pattern you are standing inside.

**Assuming CGLIB can proxy anything.** `final` classes and `final` methods defeat CGLIB. If a bean with `@PreAuthorize` is `final`, the annotation is a no-op, and the compiler will not warn you. The proxy's constraint is your constraint.

## Interview Perspective

This article is the answer to "how do these patterns show up in a real framework?" The strong answer is concrete: name `DelegatingFilterProxy` as a proxy bridging the container and the Spring context, name the filter chain as a chain of decorators over the request, name the AOP proxy as the protection proxy behind method security. A weak answer recites pattern definitions. A strong answer explains why Spring's transaction proxy exists and what it cannot do.

The interviewers probe the proxy edge cases hardest, because those are the production bugs: the self-invocation gap, the CGLIB final-class gap, the filter ordering gap. If you can explain why those gaps exist in terms of the pattern, you have proven you understand both the pattern and the framework.

Common follow-ups:

- "Why does a self-invoked `@Transactional` method not open a transaction?"
- "When does Spring use a JDK dynamic proxy and when does it use CGLIB, and what does each choice imply about your beans?"

## Knowledge Check

1. Trace a request from the container to a controller and name every structural pattern the request passes through, including the response path.
2. A `final` service class annotated with `@PreAuthorize` is injected and its method called. What happens, and what is the proxy telling you about the class?
3. A custom servlet filter is registered before Spring Security's filter and wraps the request. Explain the ordering requirement and what breaks if the wrappers collide.

## Key Takeaways

- Spring Security is a structural pattern showroom: proxies, decorators, and a chain of responsibility, none of them labeled as such.
- `DelegatingFilterProxy` and `FilterChainProxy` are proxies that bridge the container and the Spring context, and their laziness is a proxy job.
- The security filters are decorators of the request and response, which is why the servlet wrapper classes exist.
- AOP proxies are protection proxies, and the self-invocation, `final`-class, and ordering gaps are the pattern's known edges.
- Filter order is the chain's correctness, and every annotation that silently does nothing is a proxy you forgot.

## What's Next

This closes Chapter 7 on Structural Design Patterns, and Chapter 8 shifts the frame again, this time from shape to interaction. Behavioral Design Patterns are about how objects cooperate at runtime: Strategy swapping an algorithm, Observer broadcasting change, Command packaging an action, State letting an object change its behavior as it changes its state, and Chain of Responsibility, the pattern the Spring Security filter chain is made of. The mental move is to stop asking how objects are arranged and start asking how they behave and talk to each other when the system is actually running.

---

This article explains how Spring Security is built from the structural patterns, tracing a request through `DelegatingFilterProxy`, `FilterChainProxy`, and the security filter chain. It argues that the framework's magic is the proxy pattern, and that every self-invocation gap, `final`-class gap, and filter ordering bug is the proxy's known edge.
