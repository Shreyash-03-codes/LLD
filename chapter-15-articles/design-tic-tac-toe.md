# Design Tic-Tac-Toe

## Learning Objectives

- Design a system small enough that over-engineering is the actual failure mode, and learn to recognize when a pattern is ego, not engineering.
- Get the win-check right, which is the one place a tic-tac-toe design can actually be wrong.
- Learn the "n goes in, three in a row comes out" loop structure that carries the whole game, and why that loop is the deliverable.

## Introduction

Tic-tac-toe is the smallest case study in this chapter, and that is exactly why it is on the list. It has two players, a 3x3 board, and one rule: three in a row wins. There is nowhere to hide. There is no scheduling algorithm like the elevator, no state machine like the vending machine, no rule graph like chess. What is left is the core shape of every turn-based game, and the discipline of not turning a toy into a framework. Interviewers ask it early in a loop, often as a warm-up, and they use it to calibrate two things: whether you can produce a working system quickly, and whether you will bolt a visitor pattern onto a 3x3 board the moment nobody is watching.

## Requirements Gathering

Functional requirements:

- A 3x3 board starts empty.
- Two players alternate placing their mark, X and O, in an empty cell.
- The first player to complete a row, column, or diagonal of their own mark wins.
- If all nine cells fill with no winner, the game is a draw.
- The game rejects moves to occupied cells and moves out of turn.

Non-functional requirements:

- The win check runs after every move and returns instantly; it cannot require a board scan that costs more than the move itself.
- The game state is trivially serializable, because the whole point of the design is that it fits in your head.

Assumptions to state out loud: human versus human only, no AI opponent, no undo, no move history, no leaderboard, no multiple game modes. Tic-tac-toe with an AI is a minimax problem and a different interview. If you do not cut it at the start, you will spend the middle of the interview implementing the one thing the interviewer did not ask for.

## Identifying Core Entities

The entity list is three items, and it should be. This is the design where a fourth entity is the first sign of trouble.

| Entity | One-line responsibility |
| --- | --- |
| `Board` | A 3x3 grid of marks, with a method to place a mark and a method to evaluate the game outcome. |
| `Player` | A mark (X or O) and a name; the lightest possible holder of turn identity. |
| `Game` | Owns the board, alternates turns, validates moves, and reports the outcome. |

You do not need a `Cell` class, a `Position` class, or a `Move` class. A mark is a char or an enum, a cell is two integers, and a move is two integers plus the player. Every extra class here is overhead the interviewer will notice and quietly judge.

## Class Design

The board is the whole game. Model it as a 3x3 array of an enum, and give it exactly two responsibilities: place a mark if the cell is free, and evaluate the outcome after a move.

```java
public enum Mark { X, O, EMPTY }

public class Board {
    private final Mark[][] grid = new Mark[3][3];

    public Board() {
        for (int i = 0; i < 3; i++) {
            Arrays.fill(grid[i], Mark.EMPTY);
        }
    }

    public boolean isFree(int row, int col) {
        return inBounds(row, col) && grid[row][col] == Mark.EMPTY;
    }

    public void place(int row, int col, Mark mark) {
        if (!isFree(row, col)) {
            throw new IllegalArgumentException("Cell occupied");
        }
        grid[row][col] = mark;
    }

    public Mark get(int row, int col) { return grid[row][col]; }
    public boolean isFull() {
        for (Mark[] row : grid) {
            for (Mark m : row) {
                if (m == Mark.EMPTY) return false;
            }
        }
        return true;
    }
}
```

The win check is where the interview happens. The naive approach is fine: after placing a mark, check the row, the column, and, if the cell is on a diagonal, the two diagonals. Only the row and column of the last move can be completed by that move, so you check at most four lines of three. That is the trick: you do not scan the whole board, you check the lines that the last mark could possibly complete.

```java
public enum GameStatus { ONGOING, X_WINS, O_WINS, DRAW }

public class Game {
    private final Board board = new Board();
    private Mark turn = Mark.X;
    private GameStatus status = GameStatus.ONGOING;

    public GameStatus play(int row, int col) {
        if (status != GameStatus.ONGOING) {
            throw new IllegalStateException("Game over");
        }
        if (!board.isFree(row, col)) {
            throw new IllegalArgumentException("Cell occupied");
        }
        board.place(row, col, turn);
        if (isWinningMove(row, col, turn)) {
            status = turn == Mark.X ? GameStatus.X_WINS : GameStatus.O_WINS;
            return status;
        }
        if (board.isFull()) {
            status = GameStatus.DRAW;
            return status;
        }
        turn = (turn == Mark.X) ? Mark.O : Mark.X;
        return status;
    }

    private boolean isWinningMove(int row, int col, Mark mark) {
        return lineCount(row, 0, row, 1, row, 2, mark) == 3
            || lineCount(0, col, 1, col, 2, col, mark) == 3
            || (row == col && lineCount(0, 0, 1, 1, 2, 2, mark) == 3)
            || (row + col == 2 && lineCount(0, 2, 1, 1, 2, 0, mark) == 3);
    }

    private int lineCount(int r1, int c1, int r2, int c2, int r3, int c3, Mark mark) {
        int count = 0;
        if (board.get(r1, c1) == mark) count++;
        if (board.get(r2, c2) == mark) count++;
        if (board.get(r3, c3) == mark) count++;
        return count;
    }
}
```

The diagonal guards matter. `row == col` is only true on the main diagonal, and `row + col == 2` only on the anti-diagonal. A mark at (0,2) is not on the main diagonal, and a naive win check that always examines both diagonals would be correct anyway, it just does a little extra work. The guard is about clarity: you check the diagonal only when the last mark could have completed it. Either version is correct; the guarded one is the one that shows you thought about what the last move could possibly affect.

The game loop lives outside the classes, and it is the shape of the whole deliverable: print the board, ask for a move, play it, print the result. The loop is the walkthrough.

```java
Scanner sc = new Scanner(System.in);
Game game = new Game();
while (game.getStatus() == GameStatus.ONGOING) {
    System.out.println("Player " + game.getTurn() + ": enter row and col (0-2)");
    int row = sc.nextInt();
    int col = sc.nextInt();
    try {
        game.play(row, col);
    } catch (IllegalArgumentException e) {
        System.out.println("Illegal move: " + e.getMessage());
    }
    board.print();
}
System.out.println("Result: " + game.getStatus());
```

Diagram: the win check only examines the lines the last move can complete — its row, its column, and the diagonals it sits on — and the play sequence that orders win before draw.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 405" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="920" height="405" fill="#ffffff"/>

  <text x="460" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">Check only the lines the last move can complete</text>

  <g stroke="#cbd5e1" stroke-width="2">
    <line x1="120" y1="80" x2="120" y2="260"/>
    <line x1="180" y1="80" x2="180" y2="260"/>
    <line x1="60" y1="140" x2="240" y2="140"/>
    <line x1="60" y1="200" x2="240" y2="200"/>
    <rect x="60" y="80" width="180" height="180" fill="none"/>
  </g>

  <g stroke="#94a3b8" stroke-width="1.6" stroke-dasharray="6 5" fill="none">
    <line x1="150" y1="80" x2="150" y2="260"/>
    <line x1="60" y1="80" x2="240" y2="260"/>
    <line x1="60" y1="260" x2="240" y2="80"/>
  </g>

  <g stroke="#16a34a" stroke-width="4" stroke-linecap="round">
    <line x1="60" y1="180" x2="240" y2="180"/>
  </g>

  <g font-size="10.5" fill="#64748b">
    <text x="152" y="276">column</text>
    <text x="62" y="72">main diagonal</text>
    <text x="196" y="72">anti-diagonal</text>
  </g>
  <text x="252" y="184" font-size="11.5" font-weight="bold" fill="#16a34a">row 1 wins</text>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="30" font-weight="bold" text-anchor="middle">
    <text x="90" y="132" fill="#0f766e">O</text>
    <text x="210" y="132" fill="#0f766e">O</text>
    <text x="90" y="192" fill="#b91c1c">X</text>
    <text x="150" y="192" fill="#b91c1c">X</text>
    <text x="210" y="192" fill="#b91c1c">X</text>
  </g>
  <circle cx="150" cy="180" r="6" fill="#16a34a"/>
  <text x="150" y="305" text-anchor="middle" font-size="11" fill="#b91c1c" font-weight="bold">last move</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah)">
    <line x1="580" y1="162" x2="616" y2="162"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12" fill="#1f2937">
    <rect x="330" y="110" width="250" height="104" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="455" y="132" text-anchor="middle" font-weight="bold" fill="#334155">isWinningMove(row, col, mark)</text>
    <text x="455" y="154" text-anchor="middle" font-size="11" fill="#475569">its row → lineCount == 3</text>
    <text x="455" y="170" text-anchor="middle" font-size="11" fill="#475569">its column → lineCount == 3</text>
    <text x="455" y="186" text-anchor="middle" font-size="11" fill="#475569">row == col → main diagonal</text>
    <text x="455" y="202" text-anchor="middle" font-size="11" fill="#475569">row + col == 2 → anti-diagonal</text>

    <rect x="620" y="110" width="260" height="104" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="750" y="132" text-anchor="middle" font-weight="bold" fill="#14532d">Targeted, not a board scan</text>
    <text x="750" y="156" text-anchor="middle" font-size="11" fill="#166534">at most four lines of three,</text>
    <text x="750" y="172" text-anchor="middle" font-size="11" fill="#166534">only lines the last move could</text>
    <text x="750" y="188" text-anchor="middle" font-size="11" fill="#166534">possibly complete are checked</text>
    <text x="750" y="204" text-anchor="middle" font-size="11" fill="#166534">one completed line is enough</text>
  </g>

  <text x="30" y="300" font-size="14" font-weight="bold" fill="#1f2937">play() — place, win, draw, switch</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah)">
    <line x1="220" y1="343" x2="266" y2="343"/>
    <line x1="450" y1="343" x2="496" y2="343"/>
    <line x1="680" y1="343" x2="726" y2="343"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12" fill="#1f2937">
    <rect x="40" y="315" width="180" height="56" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="130" y="338" text-anchor="middle" font-weight="bold" fill="#334155">place(row, col, turn)</text>
    <text x="130" y="354" text-anchor="middle" font-size="11" fill="#64748b">rejects occupied cells</text>
    <rect x="270" y="315" width="180" height="56" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="360" y="338" text-anchor="middle" font-weight="bold" fill="#14532d">isWinningMove?</text>
    <text x="360" y="354" text-anchor="middle" font-size="11" fill="#166534">yes → X_WINS / O_WINS</text>
    <rect x="500" y="315" width="180" height="56" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="590" y="338" text-anchor="middle" font-weight="bold" fill="#92400e">isFull?</text>
    <text x="590" y="354" text-anchor="middle" font-size="11" fill="#b45309">yes → DRAW</text>
    <rect x="730" y="315" width="150" height="56" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="805" y="338" text-anchor="middle" font-weight="bold" fill="#334155">switch turn</text>
    <text x="805" y="354" text-anchor="middle" font-size="11" fill="#64748b">X ↔ O</text>
  </g>

</svg>
```

That is the entire system. Around ninety lines. Any tic-tac-toe design meaningfully larger than that is not a better design, it is a worse one that has lost the plot.

## Design Patterns Used

The correct answer is: none, and say it out loud. Tic-tac-toe has no interesting polymorphism, no state machine worth encoding, no strategy that varies. The State pattern for "whose turn" is a boolean flip, and turning that into a class hierarchy is exactly the over-engineering this case study exists to expose. The Factory pattern, the Observer pattern, a GameBuilder: all of them are ways of spending forty minutes and delivering a worse result than ninety lines of plain code. When the interviewer asks "any patterns here?", the strong answer is "no, and that is correct, because nothing here varies." Interviewers respect that answer because it means you can resist the thing their job description is full of.

## Handling Edge Cases / Concurrency

No concurrency; two humans at one terminal. The edge cases are rules, and there are exactly three worth naming. The first is playing in an occupied cell, which the `isFree` guard rejects. The second is playing after the game is over, which the status guard rejects. The third is the draw, which is not a win, and the ordering in `play` handles it correctly: win first, then full board. A full board that is also a winning board must be scored as a win, and the ordering guarantees it.

There is one design edge that catches people: what happens if a move would complete two lines at once, which happens with a mark at the center and two opposite pairs, or the final move completing a row and the diagonal simultaneously. The `isWinningMove` returns true on the first match, which is correct, because one completed line is enough to win. The status enum has no "double win" state, and that is right; tic-tac-toe has no such outcome.

## Common Mistakes

The most common mistake is the full-board scan on every move. Checking all eight lines after every move is correct and costs nothing at this scale, so it is not a performance sin. It is a design sin, because it reveals that the candidate never noticed that only the lines through the last move can change. The targeted check is the version that shows understanding.

The second mistake is the state hierarchy. `IdleState`, `XTurnState`, `OTurnState`, `GameOverState`, each a class. This is the pattern that every interview book teaches and this problem punishes. The interviewer asked for a game, not a framework. A `Mark turn` field and two guard clauses do the same job with less code and no indirection.

The third mistake is modeling the cell as a class. `Cell` with `isEmpty()`, `setMark()`, and a `Position` and a `Move`, so the 3x3 board becomes a graph of five types. None of these types carries a rule. They carry ceremony. Tic-tac-toe is the case study where you earn points by subtracting classes, not adding them.

## Interview Perspective

A weak answer is the fifty-class tic-tac-toe. The candidate has read a GoF book and is determined to use it. The board, the cells, the positions, the state classes, the observers for the display: by the time they place an X, the interviewer has seen everything except a working game.

A strong answer is the reverse: a `Board`, a `Game`, a `Mark`, and a play loop, with the win check targeted at the lines the last move can complete. When the interviewer asks "what if the move fills both diagonals", the strong candidate says "the first match wins, and that is correct because one line is enough." Follow-ups to expect: "how do you detect a draw" (isFull after a non-winning move), "what if the board were n x n" (the win check loops the row and column instead of hardcoding three, and the diagonal guards become range checks), "how do you make it two-player over a network" (the loop moves behind an API, the Game class stays exactly as it is, which is the payoff of keeping the game logic free of UI). The strongest candidates close by pointing at the total line count and saying "anything bigger than this is over-engineering." That is the answer this case study is designed to reward.

## Knowledge Check

1. The last move lands at (1,1), the center. Which lines does `isWinningMove` check, and why does it not check the row (0,1)-(1,1)-(2,1) differently from the other two cells' lines?
2. A move completes a row on the board while a separate completed column already exists from earlier moves. The winning check returns on the row. Why is the ordering of the checks, and the absence of a "double win" state, correct?
3. The board is full, no winner. Trace the exact sequence of checks in `play` that ends with `DRAW`, and explain why the full-board check must come after the win check.

## Key Takeaways

- Tic-tac-toe is ninety lines and two classes. Bigger than that is a worse design, on purpose.
- Check only the lines the last move can complete: its row, its column, and the diagonals it sits on.
- Win before draw, always. The ordering is a rule, not a style choice.
- Turn state is a field flip plus two guards, not a state class hierarchy.
- The game logic must not know about the console or the network. Keep it a plain class and the follow-ups stay cheap.

## What's Next

Tic-tac-toe proved you can keep a simple system simple. Snake and Ladder keeps the turn loop and the board but adds the element that changes the problem completely: the board is not a grid of equal squares anymore, the snakes and ladders make it a directed graph.

---

This article explains how to design tic-tac-toe as a two-class, ninety-line system where over-engineering is the actual failure mode. Its point of view is that declining to use any design pattern is the strongest answer in the interview, because nothing in the game varies.
