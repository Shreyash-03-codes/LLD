# Proxy Pattern

## Learning Objectives

- Provide a stand-in for an object that controls access, and name the three jobs a proxy actually does: defer, protect, and reach.
- Implement a lazy-loading proxy and a protection proxy in Java, and explain why the client cannot tell the proxy from the real object.
- Distinguish Proxy from Decorator by intent: Proxy controls access, Decorator adds behavior.

## Introduction

Proxy provides a surrogate for another object, so the client talks to the stand-in and never knows the real object may not exist yet, may not be reachable, or may be someone else's to touch. The proxy implements the same interface as the real object, which is what makes the substitution invisible.

The pattern has three real jobs, and they are worth naming separately because they look identical in a diagram and fail in completely different ways. A virtual proxy defers creation of an expensive object until it is actually used. A protection proxy checks permissions before letting a call through. A remote proxy stands in for an object living in another process or machine. Same shape, three very different reasons to exist.

## Problem Statement

Consider the lazy case first, because it is the one most codebases hit. A document editor loads every image when the document opens:

```java
public class Document {
    private final List<Image> images = new ArrayList<>();

    public void open(String path) {
        for (String imagePath : findImagePaths(path)) {
            images.add(new HighResImage(imagePath));
        }
    }
}
```

Opening a document with fifty images decodes all fifty at once, because `HighResImage` loads itself in its constructor. The editor stalls for seconds on open, and most of those images are below the fold and never rendered anyway. The naive fix, lazy-loading in `HighResImage` itself, couples the image class to a loading strategy, and the alternative, restructuring every caller to check "is this loaded yet," spreads the loading concern through the whole document model. The client's only real need is "give me something I can render." The timing of the load is a separate concern that has no home.

The protection case is the same shape with a different concern. A `PaymentGateway` should be callable only by authorized users, but sprinkling the authorization check through every caller guarantees somebody forgets it. Both cases have the same structure: the client should not have to worry about a cross-cutting concern, existence, permission, location, that is not its own.

## Core Concept

Proxy inserts a stand-in that owns the concern. The client sees the same interface it always saw:

```java
public interface Image {
    void render();
}
```

The real object does the expensive work:

```java
public class HighResImage implements Image {
    private final String path;

    public HighResImage(String path) {
        this.path = path;
        loadFromDisk();
    }

    private void loadFromDisk() {
        // expensive decode, seconds of work
    }

    @Override
    public void render() {
        // draw to the screen
    }
}
```

The proxy defers it:

```java
public class ImageProxy implements Image {
    private final String path;
    private HighResImage real;

    public ImageProxy(String path) {
        this.path = path;
    }

    @Override
    public void render() {
        if (real == null) {
            real = new HighResImage(path);
        }
        real.render();
    }
}
```

The document model changes one line, from `new HighResImage(path)` to `new ImageProxy(path)`, and the loading concern has a home. Nothing else in the system knows a proxy exists, because the proxy is an `Image`. When the first render happens, the image loads once and stays loaded. That is the entire virtual proxy: defer the creation, keep the interface, hide the deferral.

The protection proxy follows the same shape with a different check:

```java
public class SecuredGatewayProxy implements PaymentGateway {
    private final PaymentGateway delegate;
    private final AuthorizationService auth;

    public SecuredGatewayProxy(PaymentGateway delegate, AuthorizationService auth) {
        this.delegate = delegate;
        this.auth = auth;
    }

    @Override
    public void charge(Order order, User user) {
        if (!auth.canCharge(user, order)) {
            throw new AccessDeniedException("not authorized");
        }
        delegate.charge(order, user);
    }
}
```

Now the authorization check lives in exactly one place, and every caller gets it for free, whether they remembered to ask for it or not. The proxy does not add behavior, that would be a decorator. It guards access to behavior, and the guard is the whole job.

### The remote proxy

The remote proxy is the third job, and in Java it has mostly moved into the network stack itself. A client holds a stub that looks like the remote interface, and every call on the stub gets marshalled, shipped to a server that unmarshals it and calls the real object, and marshalled back. RMI did this for decades, and the shape survives wherever a framework exposes a local-looking API over a network boundary. The structure is the same as the virtual proxy, but the reason is distance, not laziness. The client cannot tell the object lives in another JVM, which is the point, and also the danger, because a network call hidden behind a local-looking interface fails in ways no local call ever does. If you see a proxy whose job is location, budget for the latency and the partial-failure modes the proxy hides from view.

### Dynamic proxies

Java's `Proxy` class removes the most tedious part of the pattern, writing the proxy class by hand. Hand it a classloader, an interface, and an `InvocationHandler`, and it builds the proxy at runtime, routing every method call through your handler:

```java
PaymentGateway secured = (PaymentGateway) Proxy.newProxyInstance(
        delegate.getClass().getClassLoader(),
        new Class<?>[] { PaymentGateway.class },
        (proxy, method, args) -> {
            if (!auth.canCharge(user, order)) {
                throw new AccessDeniedException("not authorized");
            }
            return method.invoke(delegate, args);
        });
```

Spring's entire AOP layer is built on exactly this: one handler that intercepts every method, checks for an aspect, and either proceeds or short-circuits. The dynamic proxy is why the pattern scales in frameworks, you write the handler once and the JVM generates a proxy per target for you. The constraint is that it only works with interfaces, which is the concrete reason Spring needs CGLIB for classes.

## Real Production Usage

Spring runs on proxies. AOP, transactions, and caching all wrap your beans in proxies: a JDK dynamic proxy when the bean implements an interface, a CGLIB subclass otherwise. When you annotate a method `@Transactional`, the bean you injected is not your class, it is a proxy that opens a transaction, calls your method, and commits or rolls back. This is the protection-and-deferral proxy at industrial scale, and it is the reason calling a transactional method from within the same class silently bypasses the transaction, you are talking to the plain object, not the proxy.

Caching is the same pattern with a different verb. `@Cacheable` wraps the method in a proxy that checks the cache and, on a miss, calls the real method and stores the result. The caching proxy is the virtual proxy's cousin, it exists to skip work rather than to guard permission. Recognizing it as a proxy predicts its edges exactly: the self-invocation gap again, a `final` method defeats it, and a cache miss behaves as if no cache existed.

Hibernate uses the proxy pattern for lazy associations. An entity's `@ManyToOne` reference is often a proxy that loads the real row from the database only when a property is first accessed. RMI was the remote proxy for decades: a stub in the client JVM that looks like the remote object and forwards calls over the network. Whenever you see a framework hand you an object that behaves like yours but quietly does something else, that something is a proxy.

## Common Mistakes

**Using a proxy where a decorator belongs, and vice versa.** The proxy controls access: creation, permission, location. The decorator adds behavior: buffering, logging, wrapping. A wrapper that adds logging and then passes the call through unchanged is a decorator. A wrapper that refuses the call unless permitted is a proxy. Mixing them up produces wrappers whose purpose is unclear, which is exactly the code that nobody dares delete.

**Forgetting the interface requirement for dynamic proxies.** `java.lang.reflect.Proxy` can only proxy interfaces, which is why Spring falls back to CGLIB for classes. Hand-rolling a proxy for a class with no interface and no `final` methods is where the pattern gets ugly, and it is often the sign that a different solution, like restructuring, would have been cheaper.

**Breaking identity without acknowledging it.** A proxy is not the real object. `proxy.equals(realObject)` is false, and a proxy wrapping an entity breaks identity-based collections. If callers rely on identity, the proxy is leaking its existence, and the deferral was not actually hidden.

## Interview Perspective

Proxy is where interviewers check whether you can separate structure from intent, because the structure is shared with Decorator and Adapter and the intent is what matters. A weak answer draws the wrapper. A strong answer says "the proxy preserves the interface and controls something: when the object is created, whether the caller is allowed, or where the call goes," and can name a real one, Spring's transactional proxy, Hibernate's lazy associations.

The follow-up that sorts people is "Proxy or Decorator?" The answer that lands names the intent of each rather than the shape, which is exactly the skill this chapter has been building.

Common follow-ups:

- "Your `@Transactional` method called from inside the same class has no transaction. Why, and what does the answer say about how Spring injects beans?"
- "A wrapper adds logging and forwards the call. Is it a Proxy or a Decorator?"

## Knowledge Check

1. In the `ImageProxy` above, what exactly changes if the proxy is removed and `HighResImage` is made lazy instead? Where does the coupling move, and who breaks?
2. Spring's transactional proxy and a protection proxy both wrap a bean. Describe one way the two proxies differ in what happens when the wrapped method throws.
3. `proxy.equals(real)` returns false. Give a concrete scenario where this matters in a production codebase that uses lazy-loaded entities.

## Key Takeaways

- Proxy is a stand-in that controls access, with three real jobs: defer creation, guard permission, reach across a boundary.
- The proxy implements the same interface, which is what hides its existence from the client.
- Proxy controls access, Decorator adds behavior, and confusing the two is how wrappers get mysterious.
- Spring, Hibernate, and RMI are the pattern in production, and their proxies are why framework magic has edges you need to know.
- Identity is the leak: a proxy is not the real object, and code that relies on identity will notice.

## What's Next

The next article is Flyweight, the last structural pattern and the most misused. Where the previous six patterns change how objects are wrapped and arranged, Flyweight changes how much state each object carries, sharing the heavy intrinsic state across many lightweight instances. We will cover the intrinsic-extrinsic split and why the pattern is genuinely rare and frequently a premature optimization.

---

This article explains the Proxy pattern as a same-interface stand-in that controls access, covering the virtual, protection, and remote jobs it does. Its central claim is that Proxy controls access while Decorator adds behavior, and that naming that intent is what separates a real proxy from a mystery wrapper.
