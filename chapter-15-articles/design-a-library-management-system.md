# Design a Library Management System

## Learning Objectives

- Model the distinction that defines every library system and every inventory system: a book title is not a physical copy, and the system must know both.
- Design the loan as the transaction record that connects a member to a copy, with due dates and fine computation as its responsibility.
- Learn the search-and-availability pattern that makes "is this book available" a question the system answers without scanning everything.

## Introduction

The library management system is the first case study in this chapter with no loop at its center. No dice to roll, no states to cycle through. Its core is data: books, members, and the loans that move copies between them, plus the rules about due dates and fines that make the loans worth tracking. Interviewers ask it because it is the cleanest example of a CRUD-heavy domain that still has real design decisions hiding in it. The biggest one is the title-versus-copy split, which most candidates miss, and which every follow-up question ("what if the library owns three copies of the same book") lands on. Get that split right and the rest of the system assembles itself.

## Requirements Gathering

Functional requirements:

- The library maintains a catalog of titles, where each title can have one or more physical copies.
- A member can search the catalog by title and author, and see which copies are available.
- A member borrows a copy; the loan records the member, the copy, and the due date.
- A member returns a copy; the loan closes and any overdue fine is calculated.
- A member can have a maximum number of active loans.

Non-functional requirements:

- Availability checks and catalog search should not require a full scan of every loan on every request.
- The system should be usable by a handful of librarians, so no optimistic concurrency theatrics beyond what a single library office needs.

Assumptions to state out loud: no reservations or holds, no inter-library loans, no member self-service kiosk, a fixed loan period and a fixed fine rate, and a single library branch (no cross-branch transfer of copies). Interviewers expect the reservation question to be cut or deferred; it is the classic scope add that turns a 45-minute design into a three-hour one. Name it, cut it, move on.

## Identifying Core Entities

The entity list is the smallest list that still carries the title-copy distinction, and that distinction is the whole game.

| Entity | One-line responsibility |
| --- | --- |
| `Book` | A title in the catalog: name, author, ISBN, and a list of copies. |
| `BookCopy` | A single physical instance, with its own ID and a current availability. |
| `Member` | A borrower with an ID, a name, and their active loans. |
| `Loan` | The transaction record: which copy, which member, borrow date, due date, return date, and fine. |
| `Library` | The facade that owns the catalog, the members, and the loan lifecycle. |
| `FineCalculator` | Computes the fine for a late return. |

If you are tempted to merge `Book` and `BookCopy` into one class, that is the design the case study exists to reject, and the follow-up question will find you out within thirty seconds.

## Class Design

`Book` is the catalog entry. It holds the metadata and the list of copies. The availability question lives here as a convenience: "do you have an available copy of this title?" delegates to the copies, which is exactly where the answer lives.

```java
public class Book {
    private final String isbn;
    private final String title;
    private final String author;
    private final List<BookCopy> copies = new ArrayList<>();

    public Book(String isbn, String title, String author) {
        this.isbn = isbn;
        this.title = title;
        this.author = author;
    }

    public BookCopy addCopy() {
        BookCopy copy = new BookCopy(isbn + "-" + (copies.size() + 1));
        copies.add(copy);
        return copy;
    }

    public Optional<BookCopy> findAvailableCopy() {
        return copies.stream().filter(c -> !c.isLoanedOut()).findFirst();
    }

    public String getTitle() { return title; }
    public String getAuthor() { return author; }
    public String getIsbn() { return isbn; }
}
```

`BookCopy` is a physical instance with exactly one meaningful field beyond its ID: whether it is currently out. The availability is boolean here, but keep it derived from the copy itself rather than from the loan, because the copy is what the member touches.

```java
public class BookCopy {
    private final String copyId;
    private boolean loanedOut;

    public BookCopy(String copyId) { this.copyId = copyId; }

    public boolean isLoanedOut() { return loanedOut; }
    public void markLoaned() { loanedOut = true; }
    public void markReturned() { loanedOut = false; }
    public String getCopyId() { return copyId; }
}
```

`Loan` is the transaction. It owns the due date and, on return, it knows how to settle itself, which means asking the fine calculator how much the member owes. Putting the fine computation behind a small collaborator keeps the "how much does a late day cost" policy from leaking into the loan record.

```java
public class Loan {
    private final BookCopy copy;
    private final Member member;
    private final LocalDate borrowedOn;
    private final LocalDate dueOn;
    private LocalDate returnedOn;

    public Loan(BookCopy copy, Member member, LocalDate borrowedOn, int loanDays) {
        this.copy = copy;
        this.member = member;
        this.borrowedOn = borrowedOn;
        this.dueOn = borrowedOn.plusDays(loanDays);
    }

    public long daysOverdue(LocalDate today) {
        return Math.max(0, ChronoUnit.DAYS.between(dueOn, today));
    }

    public long settle(LocalDate today, FineCalculator fines) {
        returnedOn = today;
        copy.markReturned();
        return fines.compute(daysOverdue(today));
    }

    public boolean isActive() { return returnedOn == null; }
    public BookCopy getCopy() { return copy; }
    public Member getMember() { return member; }
    public LocalDate getDueOn() { return dueOn; }
}
```

`FineCalculator` is a policy object. It is a whole class for what is currently one line of arithmetic, and that is deliberate: the fine rule is the single most likely thing to change, so it gets the seam. If the library later adds a cap on total fines or a grace period, `FineCalculator` changes and `Loan` does not.

```java
public class FineCalculator {
    private final long ratePerDay;

    public FineCalculator(long ratePerDay) { this.ratePerDay = ratePerDay; }

    public long compute(long daysOverdue) {
        return daysOverdue * ratePerDay;
    }
}
```

`Library` is the facade with the three operations that matter: borrow, return, search. The borrow path is where the title-copy split earns its keep, because borrowing a title requires finding a copy of that title that is not loaned out.

```java
public class Library {
    private final Map<String, Book> catalog = new HashMap<>();
    private final Map<String, Member> members = new HashMap<>();
    private final List<Loan> loans = new ArrayList<>();
    private final FineCalculator fines = new FineCalculator(50);
    private static final int MAX_ACTIVE_LOANS = 5;

    public void addBook(Book book) { catalog.put(book.getIsbn(), book); }
    public void registerMember(Member member) { members.put(member.getId(), member); }

    public Optional<Loan> borrow(String isbn, String memberId, LocalDate today) {
        Member member = members.get(memberId);
        Book book = catalog.get(isbn);
        if (member == null || book == null) {
            return Optional.empty();
        }
        long active = loans.stream().filter(l -> l.isActive() && l.getMember().equals(member)).count();
        if (active >= MAX_ACTIVE_LOANS) {
            return Optional.empty();
        }
        Optional<BookCopy> copy = book.findAvailableCopy();
        if (copy.isEmpty()) {
            return Optional.empty();
        }
        copy.get().markLoaned();
        Loan loan = new Loan(copy.get(), member, today, 14);
        loans.add(loan);
        return Optional.of(loan);
    }

    public long returnCopy(BookCopy copy, LocalDate today) {
        Loan loan = loans.stream()
                .filter(l -> l.isActive() && l.getCopy().equals(copy))
                .findFirst()
                .orElseThrow();
        return loan.settle(today, fines);
    }

    public List<Book> searchByTitle(String query) {
        return catalog.values().stream()
                .filter(b -> b.getTitle().toLowerCase().contains(query.toLowerCase()))
                .toList();
    }
}
```

Diagram: the three-record model. A title owns many physical copies, and the loan is the transaction connecting exactly one member to exactly one copy.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 860 360" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <rect width="860" height="360" fill="#ffffff"/>
  <text x="430" y="30" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">Title vs Copy vs Loan</text>

  <g stroke="#94a3b8" stroke-width="1.8" fill="none">
    <line x1="260" y1="168" x2="340" y2="95"/>
    <line x1="260" y1="176" x2="340" y2="185"/>
    <line x1="260" y1="184" x2="340" y2="275"/>
    <line x1="720" y1="130" x2="720" y2="240"/>
    <line x1="620" y1="285" x2="540" y2="185"/>
  </g>
  <g font-size="13" font-weight="bold" fill="#334155">
    <text x="266" y="155">1</text>
    <text x="300" y="250">1..*</text>
    <text x="726" y="175">1</text>
    <text x="726" y="215">0..*</text>
  </g>
  <text x="555" y="240" font-size="12.5" fill="#6b7280" text-anchor="start">records</text>
  <text x="555" y="255" font-size="12.5" fill="#6b7280" text-anchor="start">this copy</text>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="60" y="120" width="200" height="28" rx="6" fill="#3b82f6"/>
    <text x="72" y="139" font-weight="bold" fill="#ffffff">Book (title)</text>
    <rect x="60" y="148" width="200" height="82" rx="6" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="72" y="168">isbn: 978-0-13-609182-1</text>
    <text x="72" y="188">title: Clean Code</text>
    <text x="72" y="208">author: R. Martin</text>
    <text x="72" y="228">copies: [ 1..* ]</text>

    <rect x="340" y="60" width="200" height="26" rx="6" fill="#16a34a"/>
    <text x="352" y="77" font-weight="bold" fill="#ffffff">BookCopy #1</text>
    <rect x="340" y="86" width="200" height="44" rx="6" fill="#f0fdf4" stroke="#bbf7d0"/>
    <text x="352" y="110">loanedOut: false</text>

    <rect x="340" y="150" width="200" height="26" rx="6" fill="#dc2626"/>
    <text x="352" y="167" font-weight="bold" fill="#ffffff">BookCopy #2</text>
    <rect x="340" y="176" width="200" height="44" rx="6" fill="#fef2f2" stroke="#fecaca"/>
    <text x="352" y="200">loanedOut: true</text>

    <rect x="340" y="240" width="200" height="26" rx="6" fill="#dc2626"/>
    <text x="352" y="257" font-weight="bold" fill="#ffffff">BookCopy #3</text>
    <rect x="340" y="266" width="200" height="44" rx="6" fill="#fef2f2" stroke="#fecaca"/>
    <text x="352" y="290">loanedOut: true</text>

    <rect x="620" y="60" width="200" height="26" rx="6" fill="#64748b"/>
    <text x="632" y="77" font-weight="bold" fill="#ffffff">Member</text>
    <rect x="620" y="86" width="200" height="44" rx="6" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="632" y="110">id: M-42 · name: Alice</text>

    <rect x="620" y="240" width="200" height="26" rx="6" fill="#f59e0b"/>
    <text x="632" y="257" font-weight="bold" fill="#ffffff">Loan (transaction)</text>
    <rect x="620" y="266" width="200" height="64" rx="6" fill="#fffbeb" stroke="#fde68a"/>
    <text x="632" y="288">copy: BookCopy #2</text>
    <text x="632" y="308">member: M-42</text>
    <text x="632" y="328">due: borrow + 14 days</text>
  </g>

</svg>
```

The borrow method shows the ordering discipline the whole design rests on: check the member exists, check the loan limit, find an available copy, then mark and loan. Each guard is a line, and each one is a question the interviewer will ask.

## Design Patterns Used

The one pattern worth naming is the Facade, and it is a modest one: `Library` hides the catalog, the members, and the loans behind three operations, and the console or API layer never touches the maps. Beyond that, resist. The Fine Calculator is a Strategy in miniature, one interface, one implementation, and it exists for seam reasons, not pattern reasons. There is no Repository abstraction needed, because the in-memory maps are already the persistence and the interviewer knows it. If the interviewer pushes for "what about an Observer so the display updates when a book is returned," the correct answer is "the return operation returns the result value, and the UI can render it," which is the pattern-light way to say no.

## Handling Edge Cases / Concurrency

A single-branch library run by a handful of librarians is a low-concurrency system, and the honest answer is to say so, with one real exception worth naming. Two librarians can hand the same copy to two members if the find-and-mark step is not atomic. The `findAvailableCopy` and `markLoaned` pair has a check-then-act gap, and in a real system with a database, that is a row lock or an atomic `UPDATE ... WHERE loaned_out = false` on the copy. In the interview, name the gap and the fix, and then drop it; nobody expects a distributed locking lecture on a library.

The rule edges: a member at the loan limit must be refused at borrow time, not at return time; returning a copy that is not loaned out is a programming error, not a user flow, so the `orElseThrow` is honest; a fine of zero on an on-time return is the correct output of the calculator, not a bug. Each of these is a guard clause and a sentence in the walkthrough.

## Common Mistakes

The most common mistake is the merged `Book` class: one object that holds the title and a boolean `available`. The library owns three copies of a title, so one boolean cannot represent the truth, and the candidate ends up with three separate catalog entries for one book. The follow-up question "how many copies of this title do we own" becomes unanswerable. The copy list is not an optimization, it is the model.

The second mistake is treating the loan as an attribute of the member or of the copy. If the member holds a `List<Book>` and the copy holds a `dueDate`, then returning a book requires reaching into two objects and there is no single record of the transaction. The `Loan` object is the record, and the fine, the due date, and the return all live on it.

The third mistake is hardcoding the fine or the loan period into the borrow method. A `14` and a `50` appearing mid-method are policy decisions masquerading as literals. When the policy changes, the candidate has to find the literals instead of editing the policy object, and the walkthrough collapses.

## Interview Perspective

A weak answer draws `Book` with `available`, `Member` with `booksBorrowed`, and calls it done. The interviewer asks "the library has three copies of this book and two are out" and the candidate has no way to answer. There is no transaction record, so there is no way to know who has which copy or when it is due.

A strong answer opens with the split: "a title is not a copy, and a loan is a transaction between a member and a copy." Everything after that is assembly. Follow-ups to expect: "what if the library has reservations" (a queue of members per copy or per title, which changes `borrow` to `reserve` and adds a `notifyAvailable` path, and the reason we cut it at the start), "how do you search by author" (the same filter on the author field, one line), "what if a member returns a book a day late" (the calculator computes 50, the loan closes, the copy frees), "what if two copies of the same title are returned on the same day" (two loans, two settlements, two independent records; the design already handles it). The strongest candidates volunteer the check-then-act race on the copy without prompting.

## Knowledge Check

1. The library owns three copies of one title; two are loaned out. Trace `borrow` for a third member and show exactly which method returns what, and why the merged `Book.available` design cannot answer this.
2. A member returns a copy two days late with a fine rate of 50 per day. Which objects change state during `Library.returnCopy`, and in what order?
3. Two librarians simultaneously try to borrow the only remaining copy of the same title. Where is the check-then-act gap, and what is the minimal fix that closes it?

## Key Takeaways

- Title is not copy. The copy list is the model, not an optimization.
- The loan is the transaction record; it owns due date, return, and fine.
- Policy values like loan period and fine rate live in their own objects, never as literals in the flow.
- Guard in order: member exists, loan limit, available copy, then mutate.
- The check-then-act on the copy is the one concurrency story worth telling, and it is a row lock in real life.

## What's Next

The library showed you a data-centric system where the loan record is the heart. The logging framework flips the emphasis from data to behavior: it is a library of code, and the design problem is how threads hand log records to a writer without dropping any of them.

---

This article explains how to design a library management system around the split between a title and its copies, with the loan as the transaction record. It argues the loan, not the book, is the heart of the system.
