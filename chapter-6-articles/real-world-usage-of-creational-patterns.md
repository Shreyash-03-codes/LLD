# Real-World Usage of Creational Patterns

## Learning Objectives

- Recognize the five creational patterns in the JVM and in Spring and Hibernate without a diagram, by their shape rather than their name.
- Explain the distinction between framework-managed single instances and the GoF Singleton, which most engineers blur and interviewers probe.
- Judge when a real codebase should adopt a creational pattern and when it should let the framework do the creating.

## Introduction

Every article in this chapter so far has gestured at "and you will see this in Spring, and this in the JDK." This article cashes those gestures. The five creational patterns are not classroom geometry; they are the load-bearing structure of the JVM and of every serious Java framework. Learning to see them in production code is what turns a memorized catalog into a working model of how the ecosystem is put together.

The reward is not decoration. When you can say "this is a factory method" about code you are reading, you know where the seam is, what changing it will cost, and where to look for the class decision. That is comprehension, not trivia.

## Problem Statement

The failure this article addresses is the one where an engineer knows the five patterns cold, draws all five diagrams, and then cannot find any of them in a real Spring application. The textbook diagrams do not look like production code because production code rarely uses the pure form. `DocumentBuilderFactory.newInstance()` does not have a method named `factoryMethod()`. Spring's `@Bean` methods do not extend an abstract `Creator`. The pattern is present, but it is wearing different clothes.

The consequence is that engineers either conclude the patterns are academic (wrong), or they force the pure diagram onto real code and make it worse (also wrong). What is needed is the skill of translating between the diagram and the framework. That translation is exactly what this article practices.

## Core Concept

Here is the full mapping, one pattern at a time, from GoF drawing to living JVM code.

| Pattern | Pure form | Where it actually lives | What to look for |
|---------|-----------|------------------------|------------------|
| Singleton | Private constructor, static accessor | `Runtime.getRuntime()`, `Desktop.getDesktop()`, Spring singleton beans | One object, shared process-wide |
| Factory Method | Overridable `create()` in a base class | `Executors.newFixedThreadPool(...)`, `List.of(...)`, Spring `@Bean` methods | A method that returns an abstraction and hides the concrete type |
| Abstract Factory | Factory interface, one implementation per family | `DocumentBuilderFactory`, `SAXParserFactory`, Spring's `ApplicationContext` | A factory-of-factories, a coordinated stack |
| Builder | Mutable staging object, `build()` | `StringBuilder`, `HttpRequest.newBuilder()`, Guava `ImmutableList.builder()`, JPA `CriteriaBuilder` | Fluent configuration of one known class |
| Prototype | `clone()` on a registry of templates | `ArrayList.clone()`, game entity registries, Hibernate detached copy patterns | Copy instead of construct |

The pattern in each row is doing the same job in the framework that it does in the textbook, and seeing it requires noticing the shape, not the names. Take the first row, because it is the one most people get wrong.

### Singleton versus "singleton-scoped bean"

Spring beans default to singleton scope: the container creates one instance and serves it to every request. That looks exactly like the Singleton pattern, and most engineers say so. It is not. The GoF Singleton is a class that enforces its own uniqueness through a private constructor and static state. A Spring bean is an ordinary class with a normal constructor that some other object (the container) chooses to instantiate once. The difference is architectural: the GoF pattern hardcodes uniqueness into the class, which is what makes it untestable and hard to reuse; the Spring version puts uniqueness in the wiring, so the same class can be a singleton in one deployment and a new instance per request in another.

```java
@Configuration
public class AppConfig {
    @Bean
    public PaymentGateway paymentGateway() {
        return new StripeGateway(config.getApiKey());
    }

    @Bean
    @Scope("prototype")
    public ReportTemplate reportTemplate() {
        return new ReportTemplate();
    }
}
```

Same class, two scopes, decided by wiring. The GoF pattern cannot do that. This is the single most useful thing to take from this chapter into a real codebase: prefer container-managed single instances, and let the GoF Singleton stay reserved for cases with no container, like a logger or a runtime handle. Interviewers ask this exact comparison because it separates people who know patterns from people who know systems.

### Factory Method in Spring

Every `@Bean` method is a factory method. The container calls the method, gets an object, and manages its lifecycle, and the caller (`@Autowired` field, constructor parameter) never names a concrete class. That is Factory Method with the container playing the role of the creator. The same shape appears in the JDK everywhere: `Executors.newFixedThreadPool(4)` hides whether you got a `ThreadPoolExecutor` or a wrapper, and `Collections.unmodifiableList(...)` returns a type you can never name. The pattern is so common in the JDK that the GoF diagram, with its abstract creator and overriding subclass, is the rare case; the static factory method is the common one, and it is Factory Method in spirit if not in letter.

### Abstract Factory as infrastructure

`DocumentBuilderFactory.newInstance()` is Abstract Factory without the class named after it. You call a static method, get a concrete factory chosen from `META-INF/services` or system properties, and from that factory you build `DocumentBuilder` instances that all agree with each other. Spring's `ApplicationContext` is the same pattern at a much larger scale: one object is the family factory for the entire application, and its concrete implementation (XML, annotation, Groovy) determines the whole family of beans and behavior. The pattern's guarantee, that everything from one factory stays consistent, is exactly what the container is providing when it refuses to hand you beans from two different contexts.

### Builder as ergonomics

The builder is the pattern the ecosystem adopted most enthusiastically because it fixes a concrete daily annoyance. `HttpRequest.newBuilder()`, `StringBuilder`, Guava's `ImmutableList.builder()`, JPA's `CriteriaBuilder`, OkHttp's `Request.Builder`, and Lombok's generated `@Builder` all exist for one reason: named, order-independent configuration of a class that would otherwise drown in constructor arguments. When you see a fluent chain ending in `.build()`, you are looking at the pattern, full stop, no translation needed.

### Prototype as the honest exception

Prototype is the pattern the ecosystem quietly moved away from. The JDK keeps `clone()` because it must (collections clone internally), but the ecosystem's answer to copying is copy constructors, `copyOf` factories, and records, all of which copy with better guarantees. The one place the pattern thrives is where the objects are too expensive to construct or too numerous to configure by hand, and game engines and template registries remain its natural habitat. Knowing that this pattern is *rare by design* is itself knowledge, because it stops you from reaching for `clone()` where a copy constructor is the better tool.

## Real Production Usage

Two frameworks deserve a focused look because they are where the patterns stop being individual and start composing.

Spring is a factory machine built out of four of the five creational patterns. The container itself is an Abstract Factory. Its `@Bean` methods are Factory Methods. Its default bean scope gives you container-managed singletons. Its `BeanDefinitionBuilder` and `UriComponentsBuilder` are Builders. Four patterns, each doing its own job at a different layer, and none of them named after the textbook. Hibernate does the same with `SessionFactory` (an Abstract Factory for `Session` objects) and detached-entity copying that plays the Prototype role for persistence state.

The practical lesson for a codebase is more mundane and more valuable. You do not need to build any of this. The framework already constructs objects, so hand-rolled creational patterns inside a Spring application are usually redundant ceremony. The pattern work that remains is the work the container cannot do for you: domain objects with meaningful construction policy, immutable value types that need builders, and any object created outside the container's reach. The skill is knowing where your wiring ends and where your patterns begin.

## Common Mistakes

**Copying framework shapes into code that has no container.** Writing a GoF Singleton inside a Spring app duplicates what the container already does, and adds static state you cannot test. Let the container manage uniqueness; write the pattern only where the container does not reach.

**Using patterns to mirror the framework instead of the problem.** A codebase gets a `BeanFactory`-looking class because Spring has one, not because the code needs one. The framework's patterns serve the framework's needs, and yours are different.

**Equating a pattern name with its textbook implementation.** Calling `Runtime.getRuntime()` "a factory" or "a singleton" without noticing which pattern it actually is shows a diagram-based vocabulary. In production you have to recognize the shape, and the shape is what this chapter practiced.

## Interview Perspective

This is the article an interviewer is probing for when they ask "how do these patterns show up in real code?" The strong answer is not a recitation. It is the Singleton-versus-singleton-scope distinction, a concrete `@Bean` or `DocumentBuilderFactory` reference, and an honest statement about Prototype being rare. Weak answers name the pattern and describe it; strong answers name the system and show which pattern it runs on.

The interview value of this perspective is that it lets you answer "when would you use X" from experience instead of from the catalog. If you can say "I would not hand-roll a Singleton here because Spring already gives me a single instance," you have demonstrated the real competence the question was testing.

Common follow-ups:

- "Is a Spring singleton bean the Singleton pattern? Defend your answer."
- "Which of the five patterns do you see most in the JDK, and where specifically?"

## Knowledge Check

1. `Executors.newFixedThreadPool(4)` returns a type you cannot name. Name the pattern, explain what the pattern buys you here that naming the type would not, and say which textbook participant is missing.
2. A Spring `@Bean` and a GoF Singleton both yield one instance. List the three ways they differ that matter to a test suite.
3. A team hand-rolls a GoF Singleton for its cache inside a Spring application. Describe the concrete testing problem they have created and the container-native alternative.

## Key Takeaways

- The five patterns are the structure of the JVM and its frameworks, recognizable by shape rather than by name.
- Spring singleton scope is not the GoF Singleton; uniqueness in wiring is testable, uniqueness hardcoded in the class is not.
- `@Bean` methods, `Executors`, and the `List.of` family are Factory Method in production clothing.
- `DocumentBuilderFactory` and `ApplicationContext` are Abstract Factories guaranteeing family consistency.
- Prototype is rare by design in modern Java, and knowing that is as valuable as knowing the pattern itself.

## What's Next

This closes Chapter 6 on Creational Design Patterns, and the next chapter shifts the entire frame. Structural Design Patterns do not ask how objects come into existence at all; they assume the objects exist and ask how they are arranged into larger structures. Where this chapter was about construction, the next is about composition: Adapter, Facade, Decorator, Proxy, Composite, Bridge, and Flyweight all solve the problem of shape, not birth. The mental move you will need is to stop asking "how was this built" and start asking "how is this arranged and what contract does it present."

---

This article maps all five creational patterns onto the JVM, Spring, and Hibernate, teaching you to recognize them by shape instead of by diagram. Its strongest claim is that a Spring singleton bean is not the Singleton pattern at all, and that hand-rolling creational patterns inside a container usually duplicates work the framework already does.
