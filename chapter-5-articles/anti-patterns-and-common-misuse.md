# Anti-Patterns and Common Misuse

## Learning Objectives

1. Recognize the named bad solutions, the God Object, the Singleton abuse, the pattern spam, and state why each one recurs.
2. Replace a misapplied pattern with its honest alternative, the container-managed bean for the singleton, the method for the one-impl interface.
3. Apply the smell test, so you can catch an anti-pattern in review before it becomes the team's architecture.

## Introduction

The previous article taught selection. This one is the negative image: the named bad solutions that recur in every codebase that outlives its first design. An anti-pattern is the same idea as a pattern, a recurring shape with a name and consequences, except the consequences are bad. The name is just as useful, because "that's a God Object" tells a team in four words what a paragraph of review would not.

The anti-patterns that matter here are the ones the design patterns enable. Singleton, the most misused pattern in the catalog, is a factory for global state. Strategy, applied without variation, is a factory for ceremony. The patterns and the anti-patterns are the same shapes, and the difference is whether the problem they solve is real.

## Problem Statement

A payment service is built the way a lot of 2005-era Java was built. The gateway is a singleton.

```
public class Config {
    private static final Config INSTANCE = new Config();
    private Config() {}
    public static Config getInstance() {
        return INSTANCE;
    }
}

public class PaymentService {
    public Receipt charge(Order order) {
        Config config = Config.getInstance();
        ...
    }
}
```

The service reads config through a static call, and the gateway is a static field somewhere similar. The tests start failing. Every test that touches the payment path shares the one `Config` instance, so a test that changes a flag changes the world for every other test. The team cannot construct a fresh service for a test, because the service does not accept its dependencies, it reaches for them. The fix for every test is to mock a static method, and mocking a static method is the test admitting the design is welded.

That is the Singleton anti-pattern in its natural habitat. The pattern promised one instance. What it delivered was global state, a hidden dependency, and a class the tests cannot get between. It is the most expensive pattern in the catalog per line of code, and the industry spent a decade learning it by being burned by it.

## Core Concept

An anti-pattern is a named recurring solution whose consequences are negative, and the names matter because the negative consequences are shared. Naming it is the first step to removing it. The catalog that matters here has a handful of entries, and they are the ones tied to the patterns themselves.

The God Object. One class knows everything and does everything. It holds the order, the customer, the payment, the notification, and it has a method for each. It grows because every new feature finds it convenient to add a field and a method to the class that already has everything. The God Object is the outcome of never applying separation of concerns, and it is also the natural habitat of every pattern being reachable from one class. The fix is not a pattern, it is the discipline of the cohesion and responsibility articles: split the class by what changes together. The God Object is usually the result of convenience, not design, and no pattern rescues it.

Singleton abuse. The singleton gives one instance and a global accessor, and the abuse is when the global accessor becomes the team's dependency injection. The `getInstance()` call appears in every class, and every class is now welded to the singleton and to whatever the singleton itself holds. The honest replacement is the container-managed bean, Spring's default, which still gives one instance but delivers it through the constructor, so the dependency is visible and testable. The difference is not the number of instances, it is how the instance is delivered. A singleton injected is a dependency. A singleton fetched by `getInstance()` is hidden global state.

Pattern spam. The application of patterns to problems that do not have them. An interface with exactly one implementation and no prospect of a second. A factory whose only job is to construct one class. An Observer wiring that is used once, where a direct call would be clearer. Pattern spam is the selection method from the previous article, inverted: the engineer starts from the pattern and works backwards to a problem. The smell is the review comment "what is this indirection buying us," and the honest answer is usually "nothing yet."

The Golden Hammer. The pattern you know best applied to everything, which is the one-pattern version of pattern spam. The engineer who learned Strategy solves every variation problem with Strategy, and the one who learned Singleton solves every sharing problem with a singleton. The hammer is the most self-flattering anti-pattern, because it feels like expertise and is exactly the opposite.

Interface bloat. The interface that grew a method for every consumer's need, until implementing it means implementing methods you never use. It is the ISP from an earlier chapter violated at the type level, and it pairs with pattern spam, because an interface that exists for a pattern's sake is a good place for bloat to accumulate.

The misuse table, read as "when you see this, replace it with this":

| Anti-pattern | What it really is | The honest replacement |
| --- | --- | --- |
| `getInstance()` sprinkled through the code | Hidden global state and a hidden dependency | Constructor injection of a container-managed bean |
| An interface with one implementation | A diagram with no decision behind it | A concrete class until the second implementation is real |
| A factory that only builds one class | A detour | A constructor or a static method, or the container |
| A God Object | Convenience, not design | Split by what changes together |
| An Observer where the source knows its one reactor | Indirection with no decoupling | A direct call |
| Strategy with no variation | Ceremony | A method, until the variation is real |

The theme across all six is the same: the pattern is being used for its shape instead of its problem. Every anti-pattern in the table is a pattern applied where the instability sentence is false. The singleton had no reason to be global. The interface had no reason to exist. The factory had nothing to decouple. The anti-pattern is not a different kind of code, it is the same kind of code with the diagnosis missing.

The smell test is the practical tool, and it has three questions. Can I state the instability this shape is protecting? If not, the shape is decoration. Can I construct this class in a test without static mocking? If not, the dependency is hidden. Would the code be simpler with the pattern removed? If yes, remove it, and the anti-pattern was there. The three questions catch most of the catalog in review, and they are faster than remembering the six names.

The harder truth is that the anti-patterns are not always wrong. A singleton for a truly immutable config, read at startup and never mutated, is a reasonable use, the JVM itself interns strings and caches values. The difference is that the immutable config has no hidden state to corrupt tests, because there is no state. The abuse is not the one instance, it is the global mutable state and the hidden dependency. The line is drawn by testability and by whether the thing is actually global. The same test applies to every anti-pattern: the shape is an anti-pattern when its cost is being paid and its problem is not being solved.

## Real Production Usage

The production history of the anti-patterns is mostly the story of frameworks removing the reasons for them. Spring made the singleton abuse unnecessary, the bean is a singleton and it is injected, so the `getInstance()` factory vanished from a generation of codebases. The service locator, the framework-era version of the global accessor, was itself replaced by constructor injection in Spring's own evolution, which is the framework admitting the anti-pattern it had shipped. When a framework removes the reason for an anti-pattern, the codebase that still uses the anti-pattern is the one that has not moved.

The God Object still shows up in production, usually as the facade that stopped being a facade. A class starts as a narrow door to a subsystem, and a decade of features later it is the subsystem, thirty methods, twenty fields, and every feature change touches it. The fix is the same as it ever was, split by what changes together, and the pattern names help: the class that was a Facade has become the thing the Facade was supposed to hide.

The interface bloat is visible in the JDK and the ecosystem as a caution, and the contrast is instructive. `java.util.List` has stayed tight because the JDK does not add a method for every consumer. `Collection` grew a few and the authors are still paying the compatibility price. The lesson is the same at any scale: an interface pays for every method it will never remove, so adding a method to an interface is a permanent tax.

## Common Mistakes

The first mistake is treating the anti-pattern name as the diagnosis. "That's a singleton, remove it" is not a review, it is a slogan. The singleton for immutable config is fine, and the interface with one implementation is sometimes a seam worth keeping for a test double. The diagnosis is the consequence, the hidden state, the untestable dependency, the unneeded indirection, and the name is just the shortcut to finding it.

The second mistake is fixing the symptom instead of the dependency. An engineer who mocks the static gateway call to make the test pass has kept the anti-pattern and added a contortion. The honest fix moves the dependency into the constructor, and the test becomes a normal fake. The static mock is the test telling you the design is welded, and the fix is the design, not the mock.

The third mistake is overcorrecting into pattern phobia. A team burned by Singleton abuse bans all singletons, and a team burned by pattern spam bans all patterns, and both bans throw out the real problems with the anti-patterns. The abuse was never "one instance," it was hidden state. The spam was never "an interface," it was an interface with no decision. The reaction that preserves the baby is the smell test, which keeps the seam and removes the ceremony.

## Interview Perspective

"What is wrong with Singleton" is the most asked pattern question in interviews, and it is the anti-pattern question wearing a pattern's name. The weak answer is either "nothing, it's in the GoF book" or "singletons are always bad." The strong answer draws the line. "A singleton is one instance plus global access. The one instance is usually fine, Spring beans are singletons. The problem is the global access, the `getInstance()` that hides the dependency and makes the class untestable. Injected, it is a dependency. Fetched, it is hidden global state. The immutable config singleton is fine, the mutable one that every test shares is the anti-pattern."

The follow-up that tests the diagnosis is "is an interface with one implementation an anti-pattern." The weak answer says yes or no with no nuance. The strong answer applies the smell test. "It depends on why the interface exists. If it exists so a test can fake the dependency, it is a seam and it is worth it. If it exists because someone applied the pattern and the second implementation is imaginary, it is ceremony, and a concrete class would do. The interface is not the anti-pattern, the interface with no decision behind it is."

The sharper follow-up is "your reviewer says the Observer wiring is overkill, how do you respond." The strong answer runs the three-question test. "I can state the instability, adding a reactor must not touch the source, and there are two reactors now. Constructing the service takes a fake publisher, no static mocking. Removing the wiring would couple the order code to the email and the audit modules. So the wiring stays, and if it were one reactor, it would go."

## Knowledge Check

1. A `Config` class is a singleton with mutable fields, and the test suite has to reset it between tests. Name the anti-pattern, state the two consequences being paid, and write the change that removes it.

2. A teammate defends an interface with a single implementation as "good design for the future." Run the smell test on the claim and state whether the interface is a seam or ceremony, and what evidence would change your answer.

3. A God Object has grown a method for every feature for two years. State the anti-pattern and the discipline that fixes it, and name the pattern whose natural failure mode this is.

## Key Takeaways

- An anti-pattern is a pattern applied where its problem is false, and the name is the shortcut to the diagnosis.
- Singleton abuse is not the one instance, it is the hidden global state, and injection is the replacement.
- The smell test has three questions, the instability, the testability, and the simplicity, and it catches the catalog in review.
- The line between pattern and anti-pattern is drawn by consequences, not by the shape, and the immutable singleton survives while the mutable one is removed.

## What's Next

The chapter has covered the frame, the history, the map, the selection, and the failure modes. The last article turns the whole thing toward the room where it gets graded: the interview. Design Pattern Interview Strategy covers how to bring a pattern up without name-dropping, how to draw the selection method out loud, and how to say "no pattern" without looking like you do not know any.

---

This article explains the anti-patterns tied to the design patterns, the God Object, the Singleton abuse, the pattern spam, and the smell test that catches them. Its strongest claim is that an anti-pattern is a pattern applied where its problem is false, and that the immutable singleton survives review while the mutable one is removed.
