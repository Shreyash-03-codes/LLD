# Design Snake and Ladder

## Learning Objectives

- Learn to see a board game for what it structurally is, a graph, not a grid, and model the snakes and ladders as edges in that graph.
- Design a game loop that handles multiple players, turn order, and the win condition without letting any of those responsibilities leak into the board.
- Practice the "roll, move, jump, check" sequence and the edge cases that make naive implementations wrong.

## Introduction

Snake and Ladder is the case study where the number of moving parts is small and the number of ways to get them wrong is large. Dice, players, a board of a hundred cells, and a set of mappings from one cell to another. The mistake most candidates make is thinking of the board as a sequence to step through, when the game is actually a directed graph: the snakes and ladders are edges from one node to another, and a player standing on a ladder's tail does not earn the right to climb, the ladder moves them. Interviewers ask this because it is a complete turn-based game you can actually finish in forty minutes, and because the mapping structure is a perfect test of whether you model the game's data as the game actually works or as a pretty picture of a board.

## Requirements Gathering

Functional requirements:

- A board of 100 cells; a die with faces 1 through 6 is rolled each turn.
- Players alternate rolling; the player moves forward by the die value.
- If a player lands on the tail of a ladder, they are transported to its head. If they land on the head of a snake, they slide to its tail.
- A player wins when they land on or pass cell 100.
- The game supports a configurable number of players.

Non-functional requirements:

- The game must be deterministic given the dice rolls, so it is testable; the randomness must be injectable, never hardcoded inside the turn loop.
- The board configuration (snake and ladder positions) must be data, not code, so the same game class plays any board.

Assumptions to state out loud: one die, no rule about rolling a six for another turn, no rule that a player must land exactly on 100 (passing is fine), snakes cannot connect the last cell, and the board is single-dimensional with no special first-move rule. Interviewers expect you to lock these down. The "exact landing" rule and the "six gives another turn" rule are the two variants everyone argues about, so declare your cuts before the interviewer has to ask.

## Identifying Core Entities

The entity list is four items, and the interesting one is the one most candidates skip.

| Entity | One-line responsibility |
| --- | --- |
| `Board` | Holds the jump mappings and answers "what cell do I end up on from this cell?" |
| `Player` | A name, a token, and a current position. |
| `Dice` | Produces rolls; the only source of randomness in the game. |
| `Game` | Owns the turn order, the win condition, and the play loop. |

The board does not hold a grid. It holds a `Map<Integer, Integer>` from tail to head for ladders and from head to tail for snakes, and the whole game is one method: given a cell, return the cell you are actually on. That map is the entire design in miniature.

## Class Design

`Dice` is deliberately trivial, and the point of the triviality is that it can be replaced in tests. An interface with one method means the game loop never knows where randomness comes from.

```java
public interface Dice {
    int roll();
}

public class RandomDice implements Dice {
    private final Random random = new Random();
    public int roll() { return random.nextInt(6) + 1; }
}
```

`Board` owns the jumps. The key method, `resolve`, follows the mapping: you land on a cell, and if that cell is a snake or ladder tail or head, you move. The design decision worth defending is that a landing that jumps onto another snake or ladder is allowed to cascade, versus stopping after one jump. The classic rules do not cascade, so `resolve` should make exactly one jump. State that decision out loud; it is the kind of scope question interviewers probe.

```java
public class Board {
    private final int size;
    private final Map<Integer, Integer> jumps = new HashMap<>();

    public Board(int size) { this.size = size; }

    public void addSnake(int head, int tail) {
        if (tail >= head) throw new IllegalArgumentException("Snake must go down");
        jumps.put(head, tail);
    }

    public void addLadder(int tail, int head) {
        if (head <= tail) throw new IllegalArgumentException("Ladder must go up");
        jumps.put(tail, head);
    }

    public int resolve(int cell) {
        return jumps.getOrDefault(cell, cell);
    }

    public int getSize() { return size; }
    public boolean isLast(int cell) { return cell >= size; }
}
```

`Player` is a token holder. It does not roll its own dice and it does not know the board. The only rule it enforces is its own position.

```java
public class Player {
    private final String name;
    private int position = 0;

    public Player(String name) { this.name = name; }
    public String getName() { return name; }
    public int getPosition() { return position; }
    public void setPosition(int position) { this.position = position; }
}
```

`Game` is the loop. The critical detail is the order of operations in a turn: move by the roll, then resolve the jump, then check the win. Move first, jump second, win third. A naive implementation that checks the win before the jump resolves lets a player win by landing on a cell whose ladder would carry them past 100, or worse, lets a player pass the finish line in the middle of the sequence.

```java
public class Game {
    private final Board board;
    private final Dice dice;
    private final List<Player> players;
    private int turnIndex = 0;
    private boolean finished = false;

    public Game(Board board, Dice dice, List<Player> players) {
        this.board = board;
        this.dice = dice;
        this.players = players;
    }

    public TurnResult playTurn() {
        if (finished) {
            throw new IllegalStateException("Game already finished");
        }
        Player player = players.get(turnIndex);
        int roll = dice.roll();
        int landed = player.getPosition() + roll;
        int resolved = board.resolve(landed);
        player.setPosition(resolved);

        TurnResult result = new TurnResult(player, roll, landed, resolved);
        if (board.isLast(resolved)) {
            finished = true;
            result.setWinner(true);
        } else {
            turnIndex = (turnIndex + 1) % players.size();
        }
        return result;
    }
}
```

Diagram: the board as a directed graph of jumps (the board is a `Map`, not a grid), and the turn sequence where the win check runs after the jump resolves.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 470" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
    <marker id="ahG" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#16a34a"/>
    </marker>
    <marker id="ahR" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#dc2626"/>
    </marker>
  </defs>
  <rect width="920" height="470" fill="#ffffff"/>

  <text x="460" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">The board is a graph, not a grid</text>

  <text x="30" y="88" font-size="14" font-weight="bold" fill="#1f2937">Jumps as directed edges</text>

  <g stroke="#16a34a" stroke-width="2.4" fill="none" marker-end="url(#ahG)">
    <line x1="221" y1="266" x2="221" y2="196"/>
    <line x1="451" y1="266" x2="451" y2="196"/>
  </g>
  <g stroke="#dc2626" stroke-width="2.4" fill="none" marker-end="url(#ahR)">
    <line x1="313" y1="196" x2="405" y2="266"/>
    <line x1="451" y1="196" x2="129" y2="266"/>
  </g>
  <g font-size="11" fill="#166534">
    <text x="228" y="232">ladder</text>
    <text x="458" y="232">ladder</text>
  </g>
  <g font-size="11" fill="#991b1b">
    <text x="375" y="240">snake 16→8</text>
    <text x="240" y="215">snake 19→2</text>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12" fill="#1f2937">
    <g>
      <rect x="60" y="150" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="83" y="178" text-anchor="middle">11</text>
      <rect x="106" y="150" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="129" y="178" text-anchor="middle">12</text>
      <rect x="152" y="150" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="175" y="178" text-anchor="middle">13</text>
      <rect x="198" y="150" width="46" height="46" fill="#dcfce7" stroke="#16a34a"/>
      <text x="221" y="178" text-anchor="middle">14</text>
      <rect x="244" y="150" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="267" y="178" text-anchor="middle">15</text>
      <rect x="290" y="150" width="46" height="46" fill="#fee2e2" stroke="#dc2626"/>
      <text x="313" y="178" text-anchor="middle">16</text>
      <rect x="336" y="150" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="359" y="178" text-anchor="middle">17</text>
      <rect x="382" y="150" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="405" y="178" text-anchor="middle">18</text>
      <rect x="428" y="150" width="46" height="46" fill="#fee2e2" stroke="#dc2626"/>
      <text x="451" y="178" text-anchor="middle">19</text>
      <rect x="474" y="150" width="46" height="46" fill="#dcfce7" stroke="#16a34a"/>
      <text x="497" y="178" text-anchor="middle">20</text>
    </g>
    <g>
      <rect x="60" y="266" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="83" y="294" text-anchor="middle">1</text>
      <rect x="106" y="266" width="46" height="46" fill="#fee2e2" stroke="#dc2626"/>
      <text x="129" y="294" text-anchor="middle">2</text>
      <rect x="152" y="266" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="175" y="294" text-anchor="middle">3</text>
      <rect x="198" y="266" width="46" height="46" fill="#dcfce7" stroke="#16a34a"/>
      <text x="221" y="294" text-anchor="middle">4</text>
      <rect x="244" y="266" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="267" y="294" text-anchor="middle">5</text>
      <rect x="290" y="266" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="313" y="294" text-anchor="middle">6</text>
      <rect x="336" y="266" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="359" y="294" text-anchor="middle">7</text>
      <rect x="382" y="266" width="46" height="46" fill="#fee2e2" stroke="#dc2626"/>
      <text x="405" y="294" text-anchor="middle">8</text>
      <rect x="428" y="266" width="46" height="46" fill="#dcfce7" stroke="#16a34a"/>
      <text x="451" y="294" text-anchor="middle">9</text>
      <rect x="474" y="266" width="46" height="46" fill="#f8fafc" stroke="#cbd5e1"/>
      <text x="497" y="294" text-anchor="middle">10</text>
    </g>
  </g>

  <text x="30" y="370" font-size="14" font-weight="bold" fill="#1f2937">The turn — roll, move, resolve, win</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah)">
    <line x1="220" y1="410" x2="246" y2="410"/>
    <line x1="430" y1="410" x2="456" y2="410"/>
    <line x1="640" y1="410" x2="666" y2="410"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12" fill="#1f2937">
    <rect x="40" y="384" width="180" height="52" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="130" y="407" text-anchor="middle" font-weight="bold" fill="#334155">roll()</text>
    <text x="130" y="423" text-anchor="middle" font-size="11" fill="#64748b">injectable Dice interface</text>
    <rect x="250" y="384" width="180" height="52" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="340" y="407" text-anchor="middle" font-weight="bold" fill="#334155">move: position += roll</text>
    <text x="340" y="423" text-anchor="middle" font-size="11" fill="#64748b">the landing cell</text>
    <rect x="460" y="384" width="180" height="52" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="550" y="407" text-anchor="middle" font-weight="bold" fill="#1e3a8a">resolve(landed)</text>
    <text x="550" y="423" text-anchor="middle" font-size="11" fill="#1e40af">one jump, no cascade</text>
    <rect x="670" y="384" width="180" height="52" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="760" y="407" text-anchor="middle" font-weight="bold" fill="#92400e">isLast(resolved)?</text>
    <text x="760" y="423" text-anchor="middle" font-size="11" fill="#b45309">passing 100 wins</text>
  </g>

</svg>
```

`TurnResult` is the small value object that carries what happened back to the display layer. It exists because the console or the API needs to say "player 2 rolled a 5, landed on 42, slid down to 3," and the game itself does not care. Keeping the reporting payload separate from the game state is what makes the game testable and the UI replaceable.

## Design Patterns Used

The honest pattern answer here is a single one: the Strategy pattern on the dice. An interface and two implementations, real randomness and a fixed sequence for tests. That is not pattern-chasing, it is the difference between a game you can verify and a game you can only hope about. The alternative, calling `new Random()` inside `playTurn`, makes the game untestable, because there is no way to reproduce a roll. Everything else, skip. There is no Observer for the display (return the result object), no Template Method for the turn (one kind of turn exists), no Command wrapping the roll. The Game class is a plain loop and it should stay a plain loop. If the interviewer pushes toward a state machine for the turn flow, push back gently: the turn has one shape, and a state machine would encode the thing a loop already does.

## Handling Edge Cases / Concurrency

No concurrency, say so. The edge cases are the game rules, and there are three that matter. First, the ordering trap from above: roll, resolve jump, then win. A player who lands exactly on 100 is fine; a player who lands on 99 and rolls a 2 passes 100 and wins, because the requirement says passing is enough. A player who lands on a ladder whose head is 100 should win by virtue of the jump, and that only works if the win check runs after the resolution. Second, the one-jump rule: the board resolves a single mapping and returns, so a cascade never happens and you must state that this matches the classic rules. Third, the illegal-mapping guards: a snake going up or a ladder going down is a config error, and `addSnake` and `addLadder` refuse to build such a board. That guard belongs at construction time, not in the loop, because the loop should never see a board it has to second-guess.

## Common Mistakes

The most common mistake is modeling the board as a linear array and stepping a token through it, then bolting the snakes and ladders on as a lookup table that lives in `Game`. That design works, which is the problem: it works and it teaches nothing, and every follow-up question ("make the board configurable", "how do you find the minimum dice rolls to win") exposes that the graph structure was never made first-class. The jump map is not an accessory, it is the board.

The second mistake is the win check placement. Checking for a winner after adding the roll but before resolving the jump is the bug that lets a player "win" on a cell that is actually a snake head, sliding backward after the winner was declared. It is a two-line ordering fix and it is the most common correctness error in this system.

The third mistake is the injectable randomness gap. Hardcoded `new Random()` inside the turn means no test can ever reproduce a scenario. Candidates who skip the `Dice` interface are the candidates who can only say "trust me, the loop works," which is not a walkthrough.

## Interview Perspective

A weak answer is a single `SnakeLadderGame` class with the board, the players, and the dice all mixed together, and the jumps in a static array somewhere. The interviewer asks "configure this exact board" and the candidate points at a literal. The design has no seams and no answers.

A strong answer says "the board is a graph of jumps, the dice is an interface so I can test, and the turn is move, resolve, win." That sentence answers the three structural questions the case study is built on. Follow-ups to expect: "how do you compute the minimum number of rolls to win" (BFS over the board graph, where the jumps are zero-cost edges, which is the moment the graph modeling pays for itself), "what if a snake and a ladder meet at the same cell" (the resolution order is a single lookup, and the rule you declared, one jump per landing, decides it), "what if you want multiple dice" (the loop calls roll twice, the dice interface stays unchanged). The strongest candidates volunteer the BFS answer before being asked, because they already know the board is a graph and the graph has the shortest-path question written all over it.

## Knowledge Check

1. A player on cell 97 rolls a 4, landing on 101, which passes the finish line. The board's `isLast` returns true. Trace the turn and explain why passing, not exact landing, changes the placement of the win check.
2. A snake head sits at cell 95. A player on 93 rolls a 2, lands on 95, and the game declares a winner. Which line in `playTurn` is out of order, and what is the corrected sequence?
3. The board is modeled as a `Map<Integer, Integer>` of jumps. Explain how you would compute the minimum number of dice rolls to reach cell 100 from cell 0 using this model, and why the jumps behave like zero-cost edges.

## Key Takeaways

- The board is a graph: a jump map from cell to cell. Model the data as the game works, not as a picture of a board.
- Turn order: roll, move, resolve the jump, then check the win. The order is the correctness.
- Inject the dice through an interface or the game is untestable.
- Board configuration is data, validated at construction, never special-cased in the loop.
- The graph model pays for itself the moment someone asks for the shortest path, which they always do.

## What's Next

Snake and Ladder showed you a turn loop, injectable randomness, and a board that is secretly a graph. The library management system drops the dice and the turns entirely and becomes the first system in this chapter whose core is data, not a loop: books, members, and the transactions that move books between them.

---

This article explains how to design snake and ladder by modeling the board as a graph of jumps rather than linear grid. Its point of view is that the graph model is correct, not a refinement, and it turns the minimum-rolls follow-up into a BFS.
