# History and the Gang of Four

## Learning Objectives

1. Trace the lineage of design patterns from architecture and Smalltalk to the Gang of Four book, so you know why the catalog looks the way it does.
2. Argue why a 1994 book still names the shapes you draw in 2026, and which of its patterns Java has absorbed into the language and the JDK.
3. Separate the book's durable idea, named reusable solutions, from its dated furniture, the ceremony that modern Java has replaced.

## Introduction

Every pattern in this handbook traces back to one book, and it is a book most engineers have never opened. "Design Patterns: Elements of Reusable Object-Oriented Software" came out in 1994. Its authors, Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides, were known to every Java developer for decades as the Gang of Four, or just GoF, and their catalog of twenty-three patterns became the default answer to "how do I structure this."

The book is worth understanding as an object, not just a citation. It was written when C++ and Smalltalk ruled, before Java existed as a mainstream language, before the web, before microservices, before dependency injection frameworks. Yet the shapes it named are still the shapes you draw. Explaining why that is true is this article's real subject.

## Problem Statement

Ask a room of engineers "where do design patterns come from" and the answers are vague. "Some book." "The Gang of Four." Nobody can say what came before, what the book actually did, or why the list is twenty-three and not thirty. That vagueness has a cost. Engineers who treat patterns as a fixed list from a fixed book treat the list as sacred, and they miss both halves of the truth: the patterns were distilled from real codebases, and several of them are now dead weight in a language that absorbed them.

The concrete failure is the engineer who quotes the book as authority without the context. "The GoF says you need an interface and two classes, so we do it that way." The book said nothing of the kind. It described a shape that made sense in C++ in 1994, and whether the shape makes sense in your Java code depends on what Java now gives you for free. Understanding the history is what lets you argue with the book instead of worshiping it, and arguing with the book is the only way to use it well.

## Core Concept

The idea of a pattern did not start with software. It started with architecture. Christopher Alexander, a building architect, wrote about "a pattern language" in 1977, a set of named solutions to recurring problems in building design, each one a problem, a context, and a solution. "A room with a view," "a porch at the entrance." The books the software engineers read borrowed the structure and the word.

The software thread came through the Smalltalk community in the 1980s. Kent Beck and Ward Cunningham wrote about patterns for smalltalk, and James Coplien wrote about idioms for C++. The word was in the air, and a group of engineers who kept meeting, Gamma, Helm, Johnson, and Vlissides, decided the scattered ideas needed to be collected into one place. That collection became the book.

The book's method is the part worth stealing. The authors did not invent the patterns, they extracted them. They looked at well-designed object-oriented systems, including the Smalltalk and C++ libraries, found the shapes that kept recurring, and gave each one a name, a problem, a solution, and a discussion of consequences. That is why the catalog is not arbitrary: the list is an audit of what working codebases actually did, not a set of inventions.

The twenty-three patterns were organized into three families, which the next article covers in full. Creational patterns answer how objects get created without coupling the caller to the concrete class. Structural patterns answer how classes and objects compose into bigger structures. Behavioral patterns answer how objects divide responsibility and communicate. The classification was a way to organize the audit, and it has survived because it maps to real questions an engineer asks.

What the book did not have is the thing every modern reader notices first: no Java. The code was Smalltalk and C++. More importantly, the book's patterns assume a world where you must write the machinery yourself. The `Observer` pattern in the book is a hand-rolled registration and notification mechanism. The `Strategy` pattern is an interface plus classes, because there were no lambdas. The `Factory Method` is a subclass that overrides a creation hook, because there was no `Supplier`.

Modern Java absorbed a large share of the machinery. This is the shift worth understanding, because it changes which patterns you write by hand and which you get from the language.

```
// The GoF way, an anonymous class, circa Java 1.1
button.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent e) {
        handler.handle(e);
    }
});

// The same Observer pattern, absorbed into the language
button.addActionListener(e -> handler.handle(e));
```

Both snippets are the Observer pattern. The first writes the anonymous class by hand, the second lets the lambda do it. The pattern did not disappear, the ceremony did. That is the pattern of the pattern: the idea survives, the machinery gets absorbed. The JDK did the same to `Decorator` in the I/O classes, to `Iterator` as a language idiom in the enhanced for loop, to `Factory` in the `List.of`/`Set.of`/`Map.of` factories and the stream collectors.

The criticism of the book followed from the absorption. By the 2000s, a backlash argued that patterns were a sign that the language was too weak, that languages with closures and higher-order functions dissolved the patterns into syntax. The claim is half right. Some patterns, Observer, Strategy, Command, did dissolve into language features or into frameworks. Others, Singleton, were always questionable and got worse. But the structural patterns, Adapter, Facade, Proxy, Decorator, and the creational ones, Factory, Builder, Prototype, did not dissolve, because they solve composition problems that no syntax removes. The book's core idea, name the recurring shape, survived every wave of criticism because it is not a language feature, it is a way of thinking.

The history also explains the book's most famous failure. Singleton was in the catalog, and it became the most misused pattern in the industry, because engineers copied the shape, one instance, without the consequences, hidden global state and a hidden dependency. The pattern was a description of a thing that existed, not a recommendation to build it everywhere. The history is the antidote to the misuse: a pattern in the book is a report of what real code did, and a report is not a prescription.

The last piece of the history is what the book did to the industry's vocabulary. Before 1994, "Adapter" meant nothing in particular. After, it meant one specific shape, the class that translates one interface into another, and that shared meaning is what made large codebases reviewable. The twenty-three names became a technical vocabulary the way "segfault" or "deadlock" are technical vocabulary, precise, load-bearing, and worth getting right. That vocabulary, more than any single pattern, is the book's real legacy.

## Real Production Usage

The GoF patterns in production today split cleanly by whether Java absorbed them. The absorbed ones, Observer in every event listener, Strategy in every lambda passed to a method, Command in every `Runnable` and `Callable` submitted to an executor, you use daily without thinking about the book. The structural ones you still write by hand: Adapter when you wrap a third-party client, Facade when you give a subsystem a narrow door, Decorator when you stack responsibilities, Proxy when a framework intercepts calls. Spring is where the written-by-hand patterns are industrialized: `JdbcTemplate` is Template Method, the `ApplicationContext` is Factory and Singleton together, and Spring's `@Transactional` proxies are the Proxy pattern executing invisibly around your methods.

The JDK itself is the cleanest production evidence of the book's ideas surviving absorption. `java.util.Comparator` is Strategy as a functional interface. The `Iterable` interface and the enhanced for loop are Iterator turned into syntax. The `java.io` stream classes are Decorator, a stack of wrappers each adding behavior. A Java engineer who has never opened the GoF book is still a fluent user of its patterns, which is the strongest argument for why the history matters: the patterns won, and they won by becoming invisible.

## Common Mistakes

The first mistake is treating the book as law. "The GoF says" is a sentence that should always be followed by "in a C++ codebase in 1994." The book described shapes that made sense then. Whether the same shape makes sense now, in Java, with lambdas and records and a DI framework, is a separate question, and the book has no authority over it. The pattern's problem part still applies; the pattern's textbook implementation is furniture.

The second mistake is the reverse: dismissing the patterns because the book is old. "Patterns are outdated, Java has lambdas" is the mirror image of the worship. It is true for the behavioral patterns that the language absorbed and false for the structural and creational ones that no syntax replaces. The dismissal throws out the Adapter you still need because the Observer became a lambda.

The third mistake is forgetting that the patterns were extracted from code, not invented. An engineer who treats the catalog as a set of targets will force patterns into code that does not have the problem. An engineer who treats the catalog as a report of what good codebases did will reach for a pattern when the problem appears and let it alone otherwise. The difference in attitude is the difference between a collector and an engineer.

## Interview Perspective

The question "have you read the Gang of Four book" has become a trap, because the honest answer, "I know the patterns, I have not read all 400 pages," is the right answer and most candidates do not say it. The weak answer is either "yes I read it" with nothing specific, or "no" with embarrassment. The strong answer names what was actually gained. "I know the patterns from practice and the JDK, and I have skimmed the book. The idea that stuck is that the patterns were extracted from real systems, not invented, and that Java has absorbed some of them, Observer and Strategy are basically lambdas now, while the structural ones, Adapter and Decorator, are still things I write by hand."

The follow-up that tests real understanding is "why did patterns get so criticized." The weak answer repeats the slogan, "patterns are overengineering." The strong answer separates the idea from the ceremony. "The core idea, name the recurring shape, was never criticized seriously. What got criticized was the ceremony, the hand-rolled observer machinery and the Singleton abuse, and the criticism was fair, because Java and the frameworks absorbed most of the ceremony. The structural and creational patterns survived the criticism because they solve composition problems that no language feature removes."

The sharper follow-up is "which GoF pattern do you think is obsolete." The strong answer names Singleton and says why. "Singleton is the pattern the industry should have retired. It worked in the book as a description, and it became a global-variable factory in practice, with a hidden dependency and untestable state. When I need one instance, I let the container own it and inject it, which is Singleton done as Dependency Injection instead of as a `getInstance()` call."

## Knowledge Check

1. A junior engineer quotes "the GoF says Strategy needs an interface and two classes" to justify adding ceremony. State the context the quote is missing, and name the modern Java construct that replaces the concrete strategy classes.

2. Two patterns from the catalog, Observer and Adapter, have had very different fates in Java. Name which one the language absorbed and which one is still written by hand, and state the reason for the difference.

3. The GoF patterns were extracted from existing codebases, not invented. Explain why that fact changes whether the catalog is a set of targets or a set of reports, and which attitude the book's own authors took.

## Key Takeaways

- The patterns were extracted from working systems in 1994, and the list is an audit, not an invention.
- Java absorbed part of the catalog, Observer and Strategy are lambdas, Iterator is syntax, and the structural patterns survive by hand.
- The book is context, not law, and "the GoF says" should always carry its 1994 caveat.
- The real legacy is the vocabulary, twenty-three names that made large codebases reviewable.

## What's Next

The history ended with the twenty-three names and their three families. The next article is the map of those families, creational, structural, and behavioral, and it lays out the whole catalog so the selection chapters can refer to patterns by name. It is the article that turns the Gang of Four's audit into a reference you can actually use.

---

This article explains where design patterns came from, the architecture and Smalltalk lineage, the 1994 Gang of Four book, and why its twenty-three patterns still name the shapes you draw. Its strongest claim is that Java absorbed much of the catalog, so the behavioral patterns became syntax while the structural ones are still written by hand.
