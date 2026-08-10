# Design a Chess Game

## Learning Objectives

- Design a system where polymorphism is not decoration but the mechanism: a `Piece` hierarchy where every subtype supplies its own move rule.
- Model the board as the single source of truth that every move rule consults, and keep game state (whose turn, what is en passant) out of the board.
- Handle the difference between a valid move and a legal move, which is the difference between chess rules and game state.

## Introduction

Chess is the case study that separates the noun-drawers from the rule-writers. A parking lot has no rules that interact; a chess board is a dense graph of rules that all consult each other. A pawn moves one way, a bishop diagonally, and the bishop cannot move through pieces even though the board structure never tells it that. Castling involves the king, the rook, the spaces between them, and history. En passant involves the last move. Check involves the entire board and the position of one king. Interviewers ask this because it is the only classic question where the interesting design is not the classes but the move-generation and validation pipeline, and where a naive implementation fails in the most public way possible: it lets a bishop walk through a pawn.

## Requirements Gathering

Functional requirements:

- A standard 8x8 chessboard holds 32 pieces arranged in the starting position.
- Players alternate turns; white moves first.
- A piece moves according to its type's rules, cannot land on a friendly piece, cannot pass through occupied squares (except the knight), and cannot move into or leave the king in check.
- Special moves exist: castling, en passant, and pawn promotion.
- The game detects check, checkmate, and stalemate, and declares a winner or a draw.

Non-functional requirements:

- Move validation must be fast enough to run hundreds of times per second in a validation loop; no per-move setup that allocates the entire board fresh.
- The board representation should be simple enough to reason about and, if asked, easy to persist or transmit.

Assumptions to state out loud: no undo, no timers, no AI, no game clock, no save/reload mid-game, and move history exists only to answer the rules that depend on it (en passant, castling, fifty-move rule). If you do not cut AI, the interviewer gets to watch you not finish a minimax search instead of a move validator. Cut it.

## Identifying Core Entities

The entity list is short, which surprises people, because the rule complexity is hiding in the pieces.

| Entity | One-line responsibility |
| --- | --- |
| `Board` | Holds the 8x8 grid of squares and answers position queries. |
| `Piece` | A base class whose subclasses define how each type moves. |
| `PieceType` / `Color` | The type and ownership of a piece, used by the move rules. |
| `Square` | A coordinate (file, rank) plus the piece currently standing on it. |
| `Game` | Owns the board, the two players, the turn, and the checkmate and stalemate evaluation. |
| `Move` | A from-square and a to-square, plus flags for castling, promotion, and en passant. |
| `MoveValidator` | Applies the piece rules, the blocking rules, and the king-safety rule. |

The split that matters: `Piece` knows the pattern of its own movement. `MoveValidator` knows whether a move that looks like a bishop move is actually legal, which depends on the whole board and the position of the king. Pieces are dumb about the game; the validator is smart about the game.

## Class Design

Start with the coordinates and the board. `Square` is a coordinate plus whatever piece stands there. The board is an 8x8 array, and it exposes the two queries every rule needs: what is on a square, and whether a square is on the board at all.

```java
public class Square {
    private final int file; // 0-7
    private final int rank; // 0-7

    public Square(int file, int rank) { this.file = file; this.rank = rank; }
    public int getFile() { return file; }
    public int getRank() { return rank; }
}

public class Board {
    private final Piece[][] grid = new Piece[8][8];

    public boolean onBoard(Square s) {
        return s.getFile() >= 0 && s.getFile() < 8
            && s.getRank() >= 0 && s.getRank() < 8;
    }

    public Piece pieceAt(Square s) {
        return onBoard(s) ? grid[s.getFile()][s.getRank()] : null;
    }

    public void place(Piece piece, Square s) { grid[s.getFile()][s.getRank()] = piece; }
    public void remove(Square s) { grid[s.getFile()][s.getRank()] = null; }
    public void move(Square from, Square to) {
        grid[to.getFile()][to.getRank()] = grid[from.getFile()][from.getRank()];
        grid[from.getFile()][from.getRank()] = null;
    }
}
```

Now the piece hierarchy. The base class holds the color and a method that returns the candidate destinations given the board. Each subtype implements only its own movement pattern. The bishop and the rook are nearly identical: slide in directions until blocked. The knight jumps. The pawn is the weird one and deserves its own method.

```java
public abstract class Piece {
    protected final Color color;

    protected Piece(Color color) { this.color = color; }
    public Color getColor() { return color; }

    public abstract List<Square> candidateMoves(Square from, Board board);
}

public class Bishop extends Piece {
    public Bishop(Color color) { super(color); }

    public List<Square> candidateMoves(Square from, Board board) {
        List<Square> moves = new ArrayList<>();
        for (int[] dir : new int[][]{{1, 1}, {1, -1}, {-1, 1}, {-1, -1}}) {
            int f = from.getFile() + dir[0];
            int r = from.getRank() + dir[1];
            while (board.onBoard(new Square(f, r))) {
                Piece target = board.pieceAt(new Square(f, r));
                if (target != null) {
                    if (target.getColor() != color) {
                        moves.add(new Square(f, r)); // capture
                    }
                    break; // blocked beyond this
                }
                moves.add(new Square(f, r));
                f += dir[0];
                r += dir[1];
            }
        }
        return moves;
    }
}
```

The bishop shows the pattern the rook and queen copy. The key structural choice is that the blocking rule lives inside each slide, not outside. A rook copies the bishop with two directions; the queen copies with all eight. That is the entire value of the hierarchy, and it is genuine: each piece does not share the code that would otherwise be duplicated three times.

Diagram: the bishop's slide rules. Reachable squares are marked, the pawn at f6 is a capture that stops the slide, and the squares beyond it are unreachable.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 600 350" font-family="Helvetica, Arial, sans-serif">
  <g>
    <rect x="40" y="24" width="288" height="288" fill="#f0d9b5"/>
    <g fill="#b58863">
      <rect x="76" y="24" width="36" height="36"/>
      <rect x="148" y="24" width="36" height="36"/>
      <rect x="220" y="24" width="36" height="36"/>
      <rect x="292" y="24" width="36" height="36"/>
      <rect x="40" y="60" width="36" height="36"/>
      <rect x="112" y="60" width="36" height="36"/>
      <rect x="184" y="60" width="36" height="36"/>
      <rect x="256" y="60" width="36" height="36"/>
      <rect x="76" y="96" width="36" height="36"/>
      <rect x="148" y="96" width="36" height="36"/>
      <rect x="220" y="96" width="36" height="36"/>
      <rect x="292" y="96" width="36" height="36"/>
      <rect x="40" y="132" width="36" height="36"/>
      <rect x="112" y="132" width="36" height="36"/>
      <rect x="184" y="132" width="36" height="36"/>
      <rect x="256" y="132" width="36" height="36"/>
      <rect x="76" y="168" width="36" height="36"/>
      <rect x="148" y="168" width="36" height="36"/>
      <rect x="220" y="168" width="36" height="36"/>
      <rect x="292" y="168" width="36" height="36"/>
      <rect x="40" y="204" width="36" height="36"/>
      <rect x="112" y="204" width="36" height="36"/>
      <rect x="184" y="204" width="36" height="36"/>
      <rect x="256" y="204" width="36" height="36"/>
      <rect x="76" y="240" width="36" height="36"/>
      <rect x="148" y="240" width="36" height="36"/>
      <rect x="220" y="240" width="36" height="36"/>
      <rect x="292" y="240" width="36" height="36"/>
      <rect x="40" y="276" width="36" height="36"/>
      <rect x="112" y="276" width="36" height="36"/>
      <rect x="184" y="276" width="36" height="36"/>
      <rect x="256" y="276" width="36" height="36"/>
    </g>
    <g stroke="#b58863" stroke-width="1">
      <path d="M40 24 v288 M76 24 v288 M112 24 v288 M148 24 v288 M184 24 v288 M220 24 v288 M256 24 v288 M292 24 v288 M328 24 v288"/>
      <path d="M40 24 h288 M40 60 h288 M40 96 h288 M40 132 h288 M40 168 h288 M40 204 h288 M40 240 h288 M40 276 h288 M40 312 h288"/>
    </g>
  </g>

  <g fill="#16a34a">
    <circle cx="202" cy="150" r="5.5"/>
    <circle cx="130" cy="150" r="5.5"/>
    <circle cx="94" cy="114" r="5.5"/>
    <circle cx="58" cy="78" r="5.5"/>
    <circle cx="202" cy="222" r="5.5"/>
    <circle cx="238" cy="258" r="5.5"/>
    <circle cx="274" cy="294" r="5.5"/>
    <circle cx="130" cy="222" r="5.5"/>
    <circle cx="94" cy="258" r="5.5"/>
    <circle cx="58" cy="294" r="5.5"/>
  </g>

  <g stroke="#dc2626" stroke-width="2.5" fill="none">
    <circle cx="238" cy="114" r="15"/>
  </g>
  <g stroke="#9ca3af" stroke-width="1.8">
    <line x1="268" y1="72" x2="280" y2="84"/>
    <line x1="280" y1="72" x2="268" y2="84"/>
    <line x1="304" y1="36" x2="316" y2="48"/>
    <line x1="316" y1="36" x2="304" y2="48"/>
  </g>

  <circle cx="166" cy="186" r="13" fill="#ffffff" stroke="#1f2937" stroke-width="1.5"/>
  <circle cx="238" cy="114" r="12" fill="#1f2937"/>

  <g font-size="12" fill="#374151" text-anchor="middle">
    <text x="58" y="330">a</text>
    <text x="94" y="330">b</text>
    <text x="130" y="330">c</text>
    <text x="166" y="330">d</text>
    <text x="202" y="330">e</text>
    <text x="238" y="330">f</text>
    <text x="274" y="330">g</text>
    <text x="310" y="330">h</text>
  </g>
  <g font-size="12" fill="#374151" text-anchor="end">
    <text x="32" y="42">8</text>
    <text x="32" y="78">7</text>
    <text x="32" y="114">6</text>
    <text x="32" y="150">5</text>
    <text x="32" y="186">4</text>
    <text x="32" y="222">3</text>
    <text x="32" y="258">2</text>
    <text x="32" y="294">1</text>
  </g>

  <g font-size="11.5">
    <circle cx="370" cy="70" r="5.5" fill="#16a34a"/>
    <text x="392" y="74" fill="#374151">reachable square</text>
    <circle cx="370" cy="100" r="15" fill="none" stroke="#dc2626" stroke-width="2.5"/>
    <text x="392" y="104" fill="#374151">capture — blocks the slide</text>
    <g stroke="#9ca3af" stroke-width="1.8">
      <line x1="366" y1="127" x2="374" y2="135"/>
      <line x1="374" y1="127" x2="366" y2="135"/>
    </g>
    <text x="392" y="136" fill="#374151">unreachable beyond it</text>
  </g>
</svg>
```

The validator is where the rules get their teeth. It does three checks in sequence. First, the destination must be in the piece's candidate moves. Second, the move must not leave the mover's own king in check. Third, and this is the subtle one, the second check must run on a simulated board, because the whole point is to test the position after the move.

```java
public class MoveValidator {
    public boolean isLegal(Board board, Move move, Color turn) {
        Piece piece = board.pieceAt(move.getFrom());
        if (piece == null || piece.getColor() != turn) {
            return false;
        }
        if (!piece.candidateMoves(move.getFrom(), board).contains(move.getTo())) {
            return false;
        }
        // Simulate and check the king
        Board copy = simulate(board, move);
        Square kingSquare = findKing(copy, turn);
        return !isAttacked(copy, kingSquare, opposite(turn));
    }

    private Board simulate(Board board, Move move) {
        Board copy = new Board();
        // clone all 64 squares
        for (int f = 0; f < 8; f++) {
            for (int r = 0; r < 8; r++) {
                Square s = new Square(f, r);
                Piece p = board.pieceAt(s);
                if (p != null) copy.place(p.clone(), s);
            }
        }
        copy.move(move.getFrom(), move.getTo());
        return copy;
    }

    private boolean isAttacked(Board board, Square square, Color byColor) {
        for (int f = 0; f < 8; f++) {
            for (int r = 0; r < 8; r++) {
                Piece p = board.pieceAt(new Square(f, r));
                if (p != null && p.getColor() == byColor
                        && p.candidateMoves(new Square(f, r), board).contains(square)) {
                    return true;
                }
            }
        }
        return false;
    }
}
```

The simulation is the part most candidates skip, and it is the part that makes the validator actually correct. Without it, a move that exposes the king passes validation and the game is broken from then on. With it, check, checkmate, and stalemate all fall out of the same machinery: checkmate is "no legal move and my king is attacked," stalemate is "no legal move and my king is safe."

`Game` ties the turn, the history, and the end conditions together.

```java
public class Game {
    private final Board board;
    private final MoveValidator validator = new MoveValidator();
    private Color turn = Color.WHITE;
    private final List<Move> history = new ArrayList<>();

    public boolean makeMove(Move move) {
        if (!validator.isLegal(board, move, turn)) {
            return false;
        }
        board.move(move.getFrom(), move.getTo());
        history.add(move);
        turn = opposite(turn);
        return true;
    }

    public GameStatus status() {
        List<Move> legal = allLegalMoves(turn);
        boolean inCheck = validator.isKingInCheck(board, turn);
        if (legal.isEmpty()) {
            return inCheck ? GameStatus.CHECKMATE : GameStatus.STALEMATE;
        }
        return inCheck ? GameStatus.CHECK : GameStatus.ONGOING;
    }
}
```

That is the whole system in the shape that matters: piece rules, a validator that simulates, and a game that alternates turns and evaluates the end state.

## Design Patterns Used

This is one of the rare LLD cases where inheritance is the correct pattern, not a fashionable one. The `Piece` hierarchy is the Strategy pattern wearing inheritance: each piece is a move strategy selected by type. It is not dogmatically correct to complain about it, because the alternative, a giant `switch` on piece type in the validator, is objectively worse: the switch would need a case for every piece and every rule interaction, and adding a new piece type (say, a fairy chess piece) would require touching the validator. With the hierarchy, a new piece is a new class. State for the game (whose turn, the last move for en passant) is genuinely useful, but keep it in `Game` fields rather than a state class hierarchy; the chess game's turn alternation is not complex enough to need the full State pattern. Nobody credible will ding you for not wrapping turn-taking in classes.

## Handling Edge Cases / Concurrency

Chess has no concurrency, and say so plainly if asked; a chess game is two humans and one board, serialized by nature. The interesting edges are all rules. En passant: a pawn that moves two squares past an enemy pawn creates a one-move window where the enemy pawn can capture it "as if" it moved one square. The window closes after the move, which is why the `Game` keeps `lastMove` and the pawn move rule consults it. Castling: the king and rook must both be unmoved, the squares between them empty, and the king must not be in check, pass through check, or land in check. The candidate-move model handles this cleanly: the king's candidate moves include the two-square castle when the `Game` says the history allows it. Promotion: a pawn reaching the last rank must transform; the cleanest model is a `Move` flag carrying the promotion piece, applied by `Game.makeMove`. Each of these edges should be named in the interview even if not fully coded, because naming them is the evidence that you ran the walkthrough.

## Common Mistakes

The most common mistake is the giant `switch` in the validator. `switch (piece.getType()) { case PAWN: ... case KNIGHT: ... }` is the design that every rule change to one piece type forces a change to the validator, and it is the design that makes the follow-up "add a new piece type" a rewrite. The polymorphic `candidateMoves` is not a style preference; it is the property that new pieces join without touching existing code.

The second mistake is validating the destination without simulating the position. The "can I move here" check and the "does this leave my king exposed" check are different rules, and the second one requires testing a position that does not exist yet. Skipping the simulation is the bug that lets a player move into check, and from there the whole game is nonsense.

The third mistake is putting game state on the board. The board should not know whose turn it is or what the last move was. En passant and castling depend on that state, and if it lives on the board, the board has grown a memory it should not have. `Game` owns history and turn; `Board` owns position. Keep them apart.

## Interview Perspective

A weak answer is a `ChessBoard` with a `movePiece(int fromX, int fromY, int toX, int toY)` method that just swaps the array entries. There are no rules anywhere. Ask about a bishop jumping a pawn and the answer is "well, that's a separate validation step" with no step anywhere in the design.

A strong answer says "pieces know their pattern, the validator checks the pattern plus the blocking rule plus the king, and the king check runs on a simulated position." That one sentence answers the three hardest questions in the interview. Follow-ups to expect: "how do you detect checkmate" (no legal moves and king attacked, computed from the same validator), "how do you add a new piece type" (one class, its `candidateMoves` returns the pattern, nothing else changes), "how do you implement castling" (king's candidate moves consult game history, and the simulator already checks the pass-through rule). The strongest candidates volunteer the en passant window and the promotion flag without prompting, because those are the two rules every checklist forgot.

## Knowledge Check

1. A bishop and a pawn stand on the same diagonal with a single empty square between them. The pawn's move lands the bishop past the pawn. Which method produces the empty-square list, and where in that method does the pawn stop the iteration?
2. A move that is pattern-legal for the piece still gets rejected by the validator. Name the two additional checks that ran, and explain why the second one requires a simulated board.
3. The game is in stalemate: it is black's turn, black has no legal moves, and the black king is not in check. Trace how `Game.status()` distinguishes this from checkmate, and which method each branch relies on.

## Key Takeaways

- Each piece supplies its own movement pattern; the validator applies the blocking and king-safety rules. That split is the whole design.
- The king-safety check must run on a simulated board, or the validator accepts illegal moves.
- Check, checkmate, and stalemate all come from one validator: no moves plus attacked, no moves plus safe.
- Game state (turn, last move, history) lives in `Game`, never on the board.
- The hierarchy is justified because new piece types join without touching the validator. That is the test for the pattern.

## What's Next

Chess showed you rule density and the polymorphic move model. Tic-tac-toe throws out the piece hierarchy and most of the rules, leaving the win-checker and the game loop as the entire problem, and it demonstrates how to keep a simple system simple.

---

This article explains how to design a chess game by splitting piece movement patterns from the validator that applies blocking and king-safety rules. It argues the validator is only correct when it checks moves against a simulated board.
