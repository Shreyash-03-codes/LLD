# Design Splitwise

## Learning Objectives

- Model an expense as the record of who paid and who owes, with the splitting rule as a strategy, not a branch inside the expense class.
- Compute the net balance per user from a group's expenses, and learn why netting is the step every naive design forgets.
- See the debt-simplification problem for what it is, a graph problem, and know when to offer the greedy solution and what it guarantees.

## Introduction

Splitwise is the case study where the domain rules are social and the arithmetic is subtle. Friends go out, someone pays, the rest chip in, and the app's job is to decide who owes whom how much. The surface looks simple enough: record an expense, split it, show balances. The depth is in what "how much" means. A group of four people with ten expenses can owe each other in a tangle that takes twenty manual transactions to unwind, or in three, and the difference is not in the expenses, it is in the algorithm that collapses the tangle. Interviewers ask this because it is a rare case study with a real graph-theoretic question hiding in a friendly domain, and because it tests whether you can model money flowing between people without losing a cent to rounding or bias.

## Requirements Gathering

Functional requirements:

- A group has members; an expense records the payer, the total amount, and how it splits among the participants.
- Splits can be equal, exact amounts, or percentages.
- The system computes, for each member, their net balance with the group: what they are owed or owe overall.
- The system can produce the minimal or near-minimal set of settlement transactions that clears everyone's balance.

Non-functional requirements:

- Money math must be exact for the currency's smallest unit; no floating point drift, and rounding must be decided, not accidental.
- Balance computation over a group's expenses must be fast enough to run on every group view.

Assumptions to state out loud: no partial settlements outside the simplified output, no recurring expenses, no group debt to outside parties, no interest or lending, and the currency is single (no multi-currency conversion). Cut recurring expenses and cut currency. The interviewer wants the split model and the simplification, and both are clean only with those cuts.

## Identifying Core Entities

The entity list is short, and the split rules are a family, not a single class.

| Entity | One-line responsibility |
| --- | --- |
| `User` | A person in a group with a name and an ID. |
| `Expense` | The record: payer, total, participants, and the split rule that distributes it. |
| `SplitStrategy` | The rule that turns an expense total into per-participant shares. |
| `Group` | The set of users and their expenses, plus the balance computation. |
| `Balance` | A signed per-user net position: positive means owed money, negative means owes. |
| `Settlement` | A proposed transfer from one user to another that clears balances. |

The class that beginners skip is `SplitStrategy`, and skipping it is how the expense class ends up with a `switch` on split type and a hundred lines of conditionals.

## Class Design

Start with the split rule. An expense says "I paid 300, and it splits three ways." The three ways are the strategy. An interface with three implementations keeps the rule out of the expense and makes adding a fourth rule, say "split by weight," a new class instead of an edit to the switch.

```java
public interface SplitStrategy {
    List<Long> shares(long totalInCents, int participantCount);
}

public class EqualSplit implements SplitStrategy {
    public List<Long> shares(long totalInCents, int participantCount) {
        long base = totalInCents / participantCount;
        long remainder = totalInCents % participantCount;
        List<Long> result = new ArrayList<>();
        for (int i = 0; i < participantCount; i++) {
            result.add(base + (i < remainder ? 1 : 0));
        }
        return result;
    }
}

public class ExactSplit implements SplitStrategy {
    private final List<Long> exactAmounts;

    public ExactSplit(List<Long> exactAmounts) { this.exactAmounts = exactAmounts; }

    public List<Long> shares(long totalInCents, int participantCount) {
        if (exactAmounts.stream().mapToLong(Long::longValue).sum() != totalInCents) {
            throw new IllegalArgumentException("Exact split does not sum to total");
        }
        return exactAmounts;
    }
}

public class PercentSplit implements SplitStrategy {
    private final List<Integer> percents;

    public PercentSplit(List<Integer> percents) { this.percents = percents; }

    public List<Long> shares(long totalInCents, int participantCount) {
        List<Long> result = new ArrayList<>();
        long assigned = 0;
        for (int i = 0; i < participantCount; i++) {
            long share = i == participantCount - 1
                    ? totalInCents - assigned
                    : totalInCents * percents.get(i) / 100;
            assigned += share;
            result.add(share);
        }
        return result;
    }
}
```

The rounding handling is deliberate and visible. The equal split hands the remainder out one cent at a time to the first members; the percent split gives the last participant whatever is left so the shares always sum exactly to the total. Either way, the invariant is that the sum of shares equals the expense total, to the cent, always. That invariant is the whole reason to write strategies at all: rounding is a property of the strategy, not a surprise in the expense.

`Expense` is the record that joins a payer, an amount, a set of participants, and a strategy. Its one interesting method is `distribute`, which returns the per-participant shares as a map, so the group can apply them.

```java
public class Expense {
    private final String id;
    private final String payerId;
    private final long totalInCents;
    private final List<String> participantIds;
    private final SplitStrategy strategy;

    public Expense(String id, String payerId, long totalInCents,
                   List<String> participantIds, SplitStrategy strategy) {
        this.id = id;
        this.payerId = payerId;
        this.totalInCents = totalInCents;
        this.participantIds = participantIds;
        this.strategy = strategy;
    }

    public Map<String, Long> distribute() {
        List<Long> shares = strategy.shares(totalInCents, participantIds.size());
        Map<String, Long> result = new HashMap<>();
        for (int i = 0; i < participantIds.size(); i++) {
            result.put(participantIds.get(i), shares.get(i));
        }
        return result;
    }

    public String getPayerId() { return payerId; }
    public long getTotalInCents() { return totalInCents; }
}
```

`Group` computes the balances. The computation is two passes: for each expense, the payer is credited the full total, and every participant, including the payer, is debited their share. That is the step most candidates get subtly wrong: the payer participates in the split too, so a single expense both credits and debits the payer. The net is the sum of all credits minus all debits.

```java
public class Group {
    private final List<User> members;
    private final List<Expense> expenses = new ArrayList<>();

    public Group(List<User> members) { this.members = members; }

    public void addExpense(Expense expense) { expenses.add(expense); }

    public Map<String, Long> netBalances() {
        Map<String, Long> balances = new HashMap<>();
        for (Expense expense : expenses) {
            balances.merge(expense.getPayerId(), expense.getTotalInCents(), Long::sum);
            for (Map.Entry<String, Long> share : expense.distribute().entrySet()) {
                balances.merge(share.getKey(), -share.getValue(), Long::sum);
            }
        }
        return balances;
    }
}
```

The settlement computation is the graph problem. The net balances are the input: some positive, some negative, some zero. The output is a list of transfers. The greedy approach pairs the biggest creditor with the biggest debtor and cancels as much as possible, which produces a correct set of settlements and is not always the true minimum, but it is simple, fast, and what most real apps do. The candidate who can say that sentence, with the "greedy is not minimal" caveat, has the interview's deepest moment.

```java
public class SettlementCalculator {
    public List<Settlement> simplify(Map<String, Long> balances) {
        PriorityQueue<Map.Entry<String, Long>> creditors = new PriorityQueue<>(
                (a, b) -> Long.compare(b.getValue(), a.getValue()));
        PriorityQueue<Map.Entry<String, Long>> debtors = new PriorityQueue<>(
                (a, b) -> Long.compare(a.getValue(), b.getValue()));
        balances.entrySet().stream().filter(e -> e.getValue() > 0).forEach(creditors::add);
        balances.entrySet().stream().filter(e -> e.getValue() < 0).forEach(debtors::add);

        List<Settlement> settlements = new ArrayList<>();
        while (!creditors.isEmpty() && !debtors.isEmpty()) {
            Map.Entry<String, Long> creditor = creditors.poll();
            Map.Entry<String, Long> debtor = debtors.poll();
            long amount = Math.min(creditor.getValue(), -debtor.getValue());
            settlements.add(new Settlement(debtor.getKey(), creditor.getKey(), amount));
            if (creditor.getValue() - amount > 0) {
                creditor.setValue(creditor.getValue() - amount);
                creditors.add(creditor);
            }
            if (debtor.getValue() + amount < 0) {
                debtor.setValue(debtor.getValue() + amount);
                debtors.add(debtor);
            }
        }
        return settlements;
    }
}
```

Diagram: from one expense to net balances (credit the payer, debit every participant), then the settlement graph where the greedy pairing cancels the biggest balances.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 470" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
    <marker id="ahO" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#0f766e"/>
    </marker>
  </defs>
  <rect width="920" height="470" fill="#ffffff"/>

  <text x="460" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">Money flowing between people, collapsed without losing a cent</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah)">
    <line x1="250" y1="143" x2="316" y2="143"/>
    <line x1="564" y1="143" x2="630" y2="143"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12" fill="#1f2937">
    <rect x="40" y="100" width="210" height="86" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="145" y="122" text-anchor="middle" font-weight="bold" fill="#334155">Expense</text>
    <text x="145" y="142" text-anchor="middle" font-size="11" fill="#475569">A paid 300 · equal split</text>
    <text x="145" y="158" text-anchor="middle" font-size="11" fill="#475569">participants A, B, C</text>
    <text x="145" y="176" text-anchor="middle" font-size="11" fill="#475569">shares 100 each</text>

    <rect x="324" y="100" width="240" height="86" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="444" y="122" text-anchor="middle" font-weight="bold" fill="#1e3a8a">netBalances() — two passes</text>
    <text x="444" y="144" text-anchor="middle" font-size="11" fill="#1e40af">credit payer: A +300</text>
    <text x="444" y="160" text-anchor="middle" font-size="11" fill="#1e40af">debit every share: -100 each</text>
    <text x="444" y="176" text-anchor="middle" font-size="11" fill="#1e40af">the payer is a participant too</text>

    <rect x="638" y="100" width="240" height="86" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="758" y="122" text-anchor="middle" font-weight="bold" fill="#14532d">Net balances</text>
    <text x="758" y="148" text-anchor="middle" font-size="15" font-weight="bold" fill="#15803d">A +200 · B -100 · C -100</text>
    <text x="758" y="174" text-anchor="middle" font-size="11" fill="#166534">sum to zero — the proof</text>
  </g>

  <text x="30" y="238" font-size="14" font-weight="bold" fill="#1f2937">Settlements — the greedy pairing is the graph problem</text>

  <g stroke="#0f766e" stroke-width="2.2" fill="none" marker-end="url(#ahO)">
    <line x1="526" y1="290" x2="194" y2="290"/>
    <line x1="526" y1="390" x2="194" y2="290"/>
    <line x1="526" y1="390" x2="194" y2="390"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#0f766e" font-weight="bold">
    <text x="360" y="278">200</text>
    <text x="360" y="360">50</text>
    <text x="360" y="380">100</text>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <circle cx="160" cy="290" r="34" fill="#dcfce7" stroke="#16a34a"/>
    <text x="160" y="287" text-anchor="middle" font-weight="bold" fill="#14532d">A</text>
    <text x="160" y="303" text-anchor="middle" font-size="11" fill="#15803d">+250</text>
    <circle cx="160" cy="390" r="34" fill="#dcfce7" stroke="#16a34a"/>
    <text x="160" y="387" text-anchor="middle" font-weight="bold" fill="#14532d">B</text>
    <text x="160" y="403" text-anchor="middle" font-size="11" fill="#15803d">+100</text>
    <circle cx="560" cy="290" r="34" fill="#fee2e2" stroke="#dc2626"/>
    <text x="560" y="287" text-anchor="middle" font-weight="bold" fill="#7f1d1d">D</text>
    <text x="560" y="303" text-anchor="middle" font-size="11" fill="#b91c1c">-200</text>
    <circle cx="560" cy="390" r="34" fill="#fee2e2" stroke="#dc2626"/>
    <text x="560" y="387" text-anchor="middle" font-weight="bold" fill="#7f1d1d">C</text>
    <text x="560" y="403" text-anchor="middle" font-size="11" fill="#b91c1c">-150</text>
  </g>

  <text x="360" y="414" text-anchor="middle" font-size="11.5" fill="#475569">D→A 200 · C→A 50 · C→B 100</text>
  <text x="360" y="430" text-anchor="middle" font-size="11.5" fill="#475569">three transfers clear everyone</text>

</svg>
```

## Design Patterns Used

The Strategy pattern on the split is the real fit, and the reason is concrete: the three split rules are interchangeable algorithms with a shared contract, the expense does not care which one produced the shares, and adding a fourth rule is a new class. That is Strategy as the textbook intends it. Beyond that, be honest: there is no Observer for expense notifications (not in scope), no Command for creating expenses (a method call is enough), no Facade beyond a small `Group` class that earns it. The one structural idea worth naming is the two-phase balance computation, credit the payer, debit the shares, which is an accounting pattern more than a GoF one, and it is the part most designs get wrong.

## Handling Edge Cases / Concurrency

A group splitter has essentially no concurrency worth discussing, and say so. The interesting edges are arithmetic. The rounding case is the star: an equal three-way split of 100 has no whole-cent answer, and the design gives the remainder to the first members, so the shares sum to exactly 100 and no one is shorted by rounding drift. The percent split has the same problem and solves it by assigning the remainder to the last participant. The rule to state is that the sum of shares always equals the total, to the cent, and the strategies are where that invariant is enforced.

The second edge is the payer-in-participants case: the payer can be in the participant list, usually is, and the computation must debit them like everyone else. A naive implementation that splits only among the other members over-charges or under-credits, depending on the direction, and produces balances that do not sum to zero. The check "do the net balances sum to zero?" is the walkthrough's proof of correctness, and the payer handling is why it passes.

## Common Mistakes

The most common mistake is the `switch` on split type inside `Expense`. `switch (splitType) { case EQUAL: ... case EXACT: ... case PERCENT: ... }` makes every new split rule a change to the expense class, and the interviewer's "add a split by weight" becomes a hunt through the switch instead of a new strategy. The interface is not ornament, it is the extension point.

The second mistake is float money. `double amount = 300.0 / 3;` works today and breaks at some exact three-way split, because binary floating point cannot represent thirds. Integer cents and explicit remainder handling are not pedantry; they are the only way the "sum equals total" invariant survives.

The third mistake is skipping the netting step. The candidate computes pairwise debts for every expense and produces a settlement list that includes a hundred transfers. The netting pass is what collapses the tangle, and the graph simplification is the follow-up that separates the candidate who knows it from the candidate who has never heard of it.

## Interview Perspective

A weak answer is an `Expense` with `splitType`, a `HashMap` of balances, and a hardcoded equal split in the loop. The interviewer asks "how do you split 100 three ways and keep the total" and the candidate has no answer, because the rounding was never a decision. Then "how do you minimize settlements" gets silence.

A strong answer says "expenses are records, splits are strategies that enforce the sum-to-total invariant, balances are netted per user, and settlements are a greedy pairing of creditors and debtors, which is not guaranteed minimal but is correct and fast." Follow-ups to expect: "why greedy and not optimal" (the minimum-transactions problem is a known graph problem, and the greedy produces at most n minus one transactions where n is the number of nonzero balances, which is near-optimal in practice and cheap), "what if two users are in different groups" (each group computes independently, which is a scope cut we declared), "how do you add 'split by weight'" (one strategy class, zero edits elsewhere). The strongest candidates volunteer the rounding rule and the zero-sum check without prompting, because they verified their own walkthrough before presenting it.

## Knowledge Check

1. An expense of 100 is split equally among three members, one of whom is the payer. Show the three shares, then compute the group's net balances for that single expense and verify they sum to zero.
2. The greedy settlement algorithm pairs the largest creditor with the largest debtor. Give a concrete three-user example where greedy produces one more transaction than the theoretical minimum, and say what property of the input causes it.
3. The percent split of a 100 expense at 33%, 33%, and 34% produces shares that sum to 100 only because of the remainder assignment. Trace the computation for 50 split at 33%, 33%, 34% and explain why the last participant absorbs the rounding.

## Key Takeaways

- An expense is a record; the split rule is a Strategy with the invariant that shares always sum to the total.
- Integer cents everywhere, remainder handled explicitly in the strategy. Floats are a bug you cannot see.
- Net per user, two passes: credit the payer, debit every participant, and the payer is a participant.
- Settlements are the greedy creditor-debtor pairing. Correct, fast, near-minimal, and honest about not being optimal.
- The zero-sum check on net balances is the proof the whole computation is right.

## What's Next

Splitwise closed out the classic systems with a real graph problem hiding in a friendly domain. Chapter 16 opens the advanced section, and the first thing to unlearn is the frame that got you here: at the senior level, the design is no longer just classes, and what changes is bigger than the code.

---

This article explains how to design Splitwise around the expense record and the strategies that enforce the sum-to-total invariant. Its point of view is that the zero-sum check proves correctness, and the greedy settlement wins by being simple and honest about not being minimal.
