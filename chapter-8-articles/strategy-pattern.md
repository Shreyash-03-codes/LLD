# Strategy Pattern

## Learning Objectives

- Replace a growing `if/else` chain over an algorithm with a strategy interface and concrete implementations chosen at wiring time.
- Build the context so it takes the strategy in its constructor or a setter, and explain why the strategy must never know about the context.
- State the honest limit: when the variation is one `if` with no future, Strategy is overhead and you should not use it.

## Introduction

Strategy defines a family of algorithms, each behind the same interface, and lets the client pick one at runtime. The code that uses the algorithm does not know which variant it is holding, and the variant does not know who is using it. They meet through the interface and nothing else.

This is the pattern you reach for when a method keeps growing branches. Every currency, every pricing rule, every compression mode that adds another `if` to the same method is a candidate. Strategy moves those branches out of the method and gives each one a class.

## Problem Statement

Here is the failure in its native habitat. A checkout service computes delivery fees, and the fee differs by region:

```java
public class CheckoutService {
    public double deliveryFee(String region, double weight) {
        if ("US".equals(region)) {
            return 5.0 + weight * 0.5;
        } else if ("EU".equals(region)) {
            return 4.0 + weight * 0.7;
        } else if ("APAC".equals(region)) {
            return 6.0 + weight * 0.4;
        }
        return 10.0;
    }
}
```

Three regions today. Watch what happens next. A business rule arrives: EU gets a flat rate above ten kilos. That is another branch inside the `EU` branch. Then a premium-tier discount that applies per region. Now the method is a nested thicket of `if`s and no single change is safe, because every change to any region risks the fall-through of every other region. The region-specific logic, which is genuinely separate, is welded together in one method, and the method's tests have to exercise every combination to feel safe.

The deeper problem is that the method is doing two jobs. It decides which algorithm to run, and it runs it. Those are different concerns with different change rates. The decision, "this order ships to the EU," belongs with the caller, which knows the region. The execution, "compute an EU fee," belongs to the EU algorithm itself. Welding them together guarantees that every new algorithm is a new branch in one method, and that method will never stop growing.

## Core Concept

Strategy splits the two jobs. One interface names the algorithm:

```java
public interface DeliveryFeeStrategy {
    double calculate(double weight);
}
```

Each region gets a class:

```java
public class UsStrategy implements DeliveryFeeStrategy {
    @Override
    public double calculate(double weight) {
        return 5.0 + weight * 0.5;
    }
}

public class EuStrategy implements DeliveryFeeStrategy {
    @Override
    public double calculate(double weight) {
        return weight > 10.0 ? 8.0 : 4.0 + weight * 0.7;
    }
}
```

The context takes the strategy through its constructor and delegates. It keeps the decision, the wiring, out of the method entirely:

```java
public class CheckoutService {
    private final DeliveryFeeStrategy feeStrategy;

    public CheckoutService(DeliveryFeeStrategy feeStrategy) {
        this.feeStrategy = feeStrategy;
    }

    public double deliveryFee(double weight) {
        return feeStrategy.calculate(weight);
    }
}
```

The wiring moves to the composition root, the place where objects are assembled:

```java
DeliveryFeeStrategy strategy = switch (order.region()) {
    case "US" -> new UsStrategy();
    case "EU" -> new EuStrategy();
    case "APAC" -> new ApacStrategy();
    default -> new DefaultStrategy();
};
CheckoutService service = new CheckoutService(strategy);
```

Now a new region is a new class and one line in the composition root. The checkout method never changes, its tests stop growing, and each strategy is tested alone, which means the EU flat-rate rule is tested in `EuStrategyTest` without building a checkout. The context lost its branches and gained nothing in exchange except one delegate call. That is the whole value of the pattern.

Diagram: strategy interface, context, and concrete strategies

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1000 400" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="40" width="280" height="110" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="180" y="64" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">CheckoutService</text>
  <text x="180" y="92" text-anchor="middle" font-size="12" fill="#1a2733">-feeStrategy: DeliveryFeeStrategy</text>
  <text x="180" y="116" text-anchor="middle" font-size="12" fill="#1a2733">+deliveryFee(weight)</text>

  <rect x="560" y="40" width="280" height="90" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="700" y="60" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="700" y="82" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">DeliveryFeeStrategy</text>
  <text x="700" y="108" text-anchor="middle" font-size="12" fill="#1a2733">+calculate(weight)</text>

  <line x1="320" y1="90" x2="558" y2="90" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="100" y="270" width="220" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="210" y="294" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">UsStrategy</text>
  <text x="210" y="318" text-anchor="middle" font-size="12" fill="#1a2733">+calculate(weight)</text>

  <rect x="390" y="270" width="220" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="500" y="294" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">EuStrategy</text>
  <text x="500" y="318" text-anchor="middle" font-size="12" fill="#1a2733">+calculate(weight)</text>

  <rect x="680" y="270" width="220" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="790" y="294" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">ApacStrategy</text>
  <text x="790" y="318" text-anchor="middle" font-size="12" fill="#1a2733">+calculate(weight)</text>

  <line x1="210" y1="270" x2="680" y2="132" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="500" y1="270" x2="700" y2="132" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="790" y1="270" x2="720" y2="132" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
</svg>
```

The context holds the strategy and knows only the interface. The concrete strategies know nothing about the context, and the dashed arrows show they implement the interface, not that they know the checkout. One direction of knowledge, from the context to the interface, is the whole point.

### The rules that keep it honest

Two constraints matter, and they are the same constraint viewed from both sides. The strategy must not know the context. If `EuStrategy` starts reading fields off `CheckoutService`, the two are coupled and you have replaced a conditional with a leaky friendship. The strategy gets its inputs through its method parameters, and anything it needs beyond that is a sign the interface is wrong.

The context must not become a strategy router. The `switch` that picks the strategy belongs at the composition root, where objects are assembled, not inside the service. The service takes a strategy and uses it. It does not decide which strategy to construct, because the moment it does, the branches are back, wearing a new coat.

The third rule is the honest one, and it is the one most write-ups leave out. Strategy has a floor. If the variation is one `if` and there is no real chance of a second or third variant, the pattern is overhead. An interface, two classes, and a composition root are a lot of machinery to avoid one boolean. Strategy is justified by actual variation, measured or strongly predicted, and the pattern earns its cost when the branches are growing or the algorithms are independently testable. When it is not, `if (fast) { a() } else { b() }` is the better code. Refusing to apply Strategy to every branch is part of the skill.

## Real Production Usage

The JDK is the reference implementation of this pattern, and `Comparator` is the clearest instance. `Collections.sort` and `Arrays.sort` take a `Comparator`, and the sorting algorithm is fixed while the comparison strategy is supplied by the caller. Every lambda you pass to `sort` is a strategy created inline. The streams API is Strategy at scale, because the stream framework provides the pipeline and you supply every behavior: the map function, the filter predicate, the collector. The pipeline never knows what your lambdas do.

The `java.util.concurrent` package is a surprising home for it. `ThreadPoolExecutor` takes a `RejectedExecutionHandler`, which is a strategy for what happens when a task cannot be accepted, and the four built-ins, `AbortPolicy`, `CallerRunsPolicy`, `DiscardPolicy`, `DiscardOldestPolicy`, are four strategies behind one interface. You can write a fifth. That is the pattern working.

Spring uses Strategy more than any other behavioral pattern. The `PlatformTransactionManager` interface is Strategy for "how do I begin, commit, and roll back a transaction," with `DataSourceTransactionManager` and `JpaTransactionManager` as the variants, and the wiring decides which one your beans see. `RestTemplate`'s `ClientHttpRequestFactory` is Strategy for HTTP transport. The `MessageConverter` strategy behind `RestTemplate` and `@RequestBody` is how one framework handles JSON, XML, and protobuf without a single `if (format == ...)` in the framework code. When you configure Spring with a choice of two libraries, you are almost always choosing a strategy.

## Common Mistakes

**Putting the decision inside the context.** The context takes a strategy and uses it, but if it also constructs it, you have moved the branches instead of removing them. The selection belongs at the composition root, where the wiring happens once.

**Letting strategies know the context.** A strategy that reads context fields is a leak. The strategy should be stateless and dumb, receiving everything it needs as method parameters. If it needs a collaborator, pass the collaborator in, but keep the direction one-way.

**Applying Strategy to a branch that will never grow.** One currency that ships only to the US does not need an interface and a class. The pattern is overhead until there are at least two real variants with independent lifecycles, and the test to apply is whether the algorithm changes independently of the code that calls it.

## Interview Perspective

Strategy is the behavioral pattern interviewers start with, because it is simple to draw and hard to apply well. A weak answer defines the interface and shows two implementations. A strong answer explains why the decision moved out of the method, where the decision now lives, the composition root, and why the strategy must stay dumb, which is the part that separates people who have maintained one from people who have drawn one.

The follow-up that sorts people is the one about the refactor. "This method has a boolean that switches between two algorithms. What do you do?" The weak answer says "make a Strategy interface." The strong answer asks what the boolean means before touching anything, because a boolean that picks an algorithm is Strategy, but a boolean that changes behavior over time is State, and the two are not interchangeable. The skill being tested is diagnosis, not application.

Common follow-ups:

- "Where does the `switch` that picks the strategy live, and why?"
- "Your strategy needs a collaborator it cannot get from its parameters. What are your options?"

## Knowledge Check

1. Refactor the three-region `deliveryFee` method into strategies, then add a flat-rate EU rule and show which files change.
2. `ThreadPoolExecutor` takes a `RejectedExecutionHandler`. Name the context, the strategy interface, and two built-in strategies, and describe a scenario where you would write a custom one.
3. A method has exactly one `if/else` over an algorithm and no predicted growth. Argue for leaving it as a conditional, and state the condition under which you would change your mind.

## Key Takeaways

- Strategy removes algorithm branches by moving each variant behind one interface and choosing it at the wiring point.
- The strategy stays dumb and stateless, the context stays decisive and delegation-only, and the direction of knowledge is one-way.
- The decision belongs at the composition root, never inside the context.
- `Comparator`, `RejectedExecutionHandler`, and `PlatformTransactionManager` are Strategy in the JDK and Spring.
- Strategy is overhead until the variation is real, and refusing to apply it to a one-branch method is correct.

## What's Next

The next article is Observer, the pattern you reach for when a change has to reach many interested parties and you do not want every party hardcoded into the thing that changed. Where Strategy removes branches over an algorithm, Observer removes branches over who gets told. We will cover the subject-observer contract, the push-versus-pull trade, and why this pattern is both the most useful and the most overused in the chapter.

---

This article explains the Strategy pattern as moving algorithm branches behind an interface and choosing the variant at the wiring point. It argues the strategy must stay dumb and the decision must live at the composition root.
