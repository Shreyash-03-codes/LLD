# Factory vs Builder: When to Use Which

## Learning Objectives

- State the real difference in one sentence: a factory decides *which* class you get, a builder decides *how one class* gets assembled.
- Apply a decision procedure under interview pressure without defaulting to the pattern you like more.
- Recognize the code smell where someone implemented a factory as a method with six optional parameters, which is a Builder in disguise.

## Introduction

These two are the most-confused pair in the creational catalog, and the confusion is understandable because both live next to the word "create" and both can return the same type. The mistake people make is treating them as competing solutions to the same problem. They are not. They answer different questions, they fail differently, and the right choice falls out of a single question you can ask about the code in front of you.

The question is not "is this construction complex?" It is "who decides, and what exactly do they decide?"

## Problem Statement

Here is the failure that happens when you pick by vibes instead of by the question. A team needs a `Parser` for uploaded files. The first engineer writes a factory:

```java
public class ParserFactory {
    public Parser create(String type, boolean lenient,
                         int bufferSize, Locale locale,
                         boolean useCache, String encoding) {
        return switch (type) {
            case "csv" -> new CsvParser(lenient, bufferSize, locale, useCache, encoding);
            case "json" -> new JsonParser(lenient, bufferSize, locale, useCache, encoding);
            default -> throw new IllegalArgumentException("unknown type: " + type);
        };
    }
}
```

Look at that signature. Six parameters, four of which are options that every parser happens to share. The factory is doing two jobs: it is choosing a concrete class (a factory's job) and it is collecting a pile of configuration that the caller does not want to think about (a builder's job). The second job is leaking. Every caller now has to supply `lenient`, `bufferSize`, and `useCache` for every call even when the defaults are fine, and the parameter order is already wrong in someone's code. The next field, say `maxDepth`, makes it seven. This is what a factory looks like when it was born on the wrong side of the line.

The symmetrical failure is a builder that hides the class decision. Someone writes a fluent chain that ends by switching internally:

```java
Report report = Report.builder()
        .source("monthly")
        .format("pdf")
        .template(invoiceTemplate)
        .build();
```

and inside `build()` the `source` field decides the concrete report class. Now the builder is a factory wearing a fluent costume, and every caller thinks it is assembling one class when it is actually routing. Both directions hurt, and the fix for both is the same: split the two jobs.

## Core Concept

Factory Method and Builder sit on different axes, and the whole chapter has been building toward this one distinction:

| | Factory Method | Builder |
|---|---|---|
| The question it answers | Which concrete class do I get? | How is this one class assembled? |
| What varies | The type behind the abstraction | The combination of values inside a known type |
| The thing the caller does not know | The concrete class | The valid construction sequence |
| Fails when | You add a variant and forget a branch | You add a field and miss a copy site |
| Typical smell when misused | Factory signature grows optional params | Builder ends with a type switch inside `build()` |

The one-sentence version you can keep in your pocket: **a factory hides which class you get, a builder hides how one class is put together.** If you can already name the concrete class you want, you cannot be looking for a factory. If the class is already known but the constructor is an unreadable pile of arguments, you cannot be looking for a builder.

Two concrete examples make it land. Builder territory:

```java
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com"))
        .header("Accept", "application/json")
        .timeout(Duration.ofSeconds(5))
        .build();
```

Every caller knows it wants an `HttpRequest`. The variability is in the six optional settings, and the builder lets each call site set only what it cares about. A factory here would have to either enumerate every combination or accept the six-parameter signature we already saw fail.

Factory territory:

```java
Parser parser = ParserFactory.forFile(Path.of("report.csv"));
List<Record> records = parser.parse(path);
```

Here the caller genuinely does not know, and should not know, whether `report.csv` yields a `CsvParser` or something else. The file extension decides. That decision is the factory's entire job, and the factory signature stays at one argument because there is nothing else for the caller to configure.

### The decision procedure

Under interview pressure, or at a keyboard, run these in order:

1. Does the caller know the concrete class it wants? If no, it is a factory.
2. Does the class have many optional, interdependent, or easily-misordered fields? If yes, it is a builder.
3. Does the answer differ across callers for the same logical object? If the caller knows the class but different callers want different field combinations, it is still a builder, and the factory-shaped alternative (a factory with a growing parameter list) is the exact thing you are trying to avoid.
4. Do you need a *family* of objects to stay consistent? That exits this comparison entirely and sends you to Abstract Factory.

Step 2 deserves the blunt version. A factory method whose parameters are mostly optional configuration is not a factory, it is a builder with a worse API and no validation home. The parameter list of your factory is the diagnostic: if it reads like a settings object, you wanted a builder, and the settings object is the builder's natural staging area.

A related but distinct trap is the "factory of builders." It is real and legitimate: you can have a `ParserFactory` whose `create()` returns a `CsvParser` configured by a small builder when the parser's options are themselves numerous. That is composition, not confusion, and it is the normal resolution when an object has both a family axis and a configuration axis. Factory for the family, builder for the settings. They coexist because they answer different questions.

### Why the boundary matters in practice

The cost of getting it wrong is not aesthetic. A factory used as a builder forces every call site to know about options it does not care about, and it grows a parameter list that accumulates cruft nobody removes. A builder used as a factory hides a branch inside `build()`, which means the thing that decides your class is buried behind a chain of setters and invisible at every call site. Both are failures of the same kind: the code hid the wrong decision. The factory hides type selection correctly. The builder hides assembly correctly. Use each to hide exactly what it is good at.

## Real Production Usage

The JVM gives you both, side by side, so you can study the line in the wild. `java.net.http.HttpRequest.newBuilder()` is pure builder: the class is fixed, the configuration is what varies. `java.util.concurrent.Executors.newFixedThreadPool(...)` is pure factory: the concrete `ExecutorService` implementation is hidden. The standard library draws the boundary cleanly, which is why it is the best reference for the distinction.

Guava is the other good study: `ImmutableList.builder()` is builder territory, while `ImmutableList.copyOf(...)` and the static `of(...)` factories choose between full copies, empty lists, and singleton lists behind one name. Same library, both patterns, each on the correct side of the line.

## Common Mistakes

**The factory with six optional parameters.** As above, a factory signature that reads like a settings object. It forces callers to pass configuration they do not have opinions about. Refactor the options into a builder or a config object and keep the factory choosing the type only.

**A builder whose `build()` decides the concrete class.** A fluent chain that routes to different types internally. Every caller believes it is assembling one class, and the type selection is invisible. If the class varies, the decision belongs in a factory at the top, not a switch at the bottom.

**Fixing a confused design by adding the other pattern on top.** Wrapping a bad factory in a builder, or adding a factory that wraps a bad builder, just layers ceremony over a decision that is still in the wrong place. Decide which decision you are actually trying to hide, then remove the wrong layer.

## Interview Perspective

This comparison is a favorite because it separates people who learned patterns from diagrams from people who have debugged them. A weak answer is a memorized list of differences. A strong answer is the one-sentence rule plus the diagnostic: "If I can name the concrete class, it is not a factory problem. If the class is fixed but the constructor is a mess, it is not a builder problem."

The follow-up questions usually force the decision procedure, so having the four steps ready matters more than knowing definitions.

Common follow-ups:

- "Your factory keeps growing parameters. What does that tell you, and what do you do?"
- "Can they be combined? Draw me the factory-of-builders shape and justify it."

## Knowledge Check

1. A `Report` has three renderers (`pdf`, `html`, `text`) and each takes roughly ten options that share defaults. Classify the two axes and sketch the factory-of-builders shape that serves both.
2. Your `parse()` method on `CsvParser` behaves differently depending on a `lenient` flag. Should `lenient` arrive via factory parameter or builder method, and why does the choice change how callers evolve?
3. Someone argues "both patterns return objects, so the difference is cosmetic." Give one change that is cheap under the right pattern and expensive under the wrong one.

## Key Takeaways

- Factory hides which class, Builder hides how one class is assembled, and that axis is the whole distinction.
- A factory with a growing list of optional parameters is a Builder in disguise.
- A builder that switches on a field inside `build()` is a Factory in disguise.
- Run the decision procedure in order: unknown class first, then many optional fields, then family consistency.
- When an object has both axes, compose them: factory for the family, builder for the settings.

## What's Next

The final article of this chapter steps back from the five patterns to look at the whole family in the wild. Real-World Usage of Creational Patterns maps each pattern to the JVM and ecosystem frameworks that actually run on it, and it settles how Spring, Hibernate, and the JDK distribute these patterns across real systems. After that we cross into the next chapter, Structural Design Patterns, and the shift will be obvious: where creational patterns control *what objects exist*, structural patterns control *how existing objects are arranged into larger structures*.

---

This article explains how to tell Factory and Builder apart by asking what decision the code hides, not by memorizing diagrams. It argues that a factory whose parameter list grows with optional settings is a builder in disguise, and that the correct resolution when an object has both axes is to compose the two patterns rather than fuse them.
