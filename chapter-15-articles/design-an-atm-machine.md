# Design an ATM Machine

## Learning Objectives

- Learn the split that defines this case study: the ATM is a dumb terminal and the bank owns the money, and why putting the balance on the ATM breaks everything.
- Model a session as a sequence of validated steps: card, PIN, account selection, then operations.
- Reason about the one genuinely interesting concurrency problem in LLD, two ATMs withdrawing from the same account at the same moment.

## Introduction

Every LLD chapter has a case study whose real lesson is about ownership. For the ATM, the lesson is that the machine in the lobby does not know how much money is in your account. The moment a candidate draws an `Account` class with a `balance` field on the ATM, they have built a system that works on paper and fails in the world: two ATMs, one account, two withdrawals, and the balance is wrong twice. The ATM is a front end to a bank, and the bank's database is the only thing that decides what a withdrawal means. Interviewers ask this question because it is the cleanest way to test whether you understand where the source of truth lives in a system with a client and a server pretending to be one device.

## Requirements Gathering

Functional requirements:

- A user inserts a card and authenticates with a PIN.
- After authentication, the user can withdraw cash, deposit cash or checks, check balances, and transfer between accounts.
- The machine dispenses cash from its own cassette; it can only dispense what it physically holds and in denominations it can make.
- The user ends the session by taking the card back.

Non-functional requirements:

- No single transaction can double-spend a balance, even under concurrent access from multiple ATMs.
- The ATM must never dispense more cash than it holds or more than the account allows.
- Failed and successful transactions are recorded for reconciliation.

Assumptions to state out loud: the bank owns the accounts and the ATM talks to it over a network; the ATM does not cache balances; there is no network partition handling beyond "transaction fails and rolls back"; card is physical, so no card-less or mobile withdrawal flow. Cut card-less withdrawal. Cut cash recycling. If you do not cut those, you will be modeling QR codes when the interviewer wanted to talk about a deadlock.

## Identifying Core Entities

The entity list divides cleanly into the machine side and the bank side.

| Entity | One-line responsibility |
| --- | --- |
| `ATM` | The terminal: authenticates, takes transaction intents, dispenses from its cassette. |
| `Card` | The identifier presented at the start of a session. |
| `ATMSession` | Tracks the authenticated state: which card, which account, PIN verified. |
| `Bank` | The source of truth for accounts; processes transactions atomically. |
| `Account` | A balance and an account number, owned by the bank. |
| `Transaction` | A record of a withdrawal, deposit, or transfer, produced by a successful bank operation. |
| `CashDispenser` | The physical cassette that reports how much cash remains. |

The ownership split is visible in the list itself. `Account` sits next to `Bank`, not next to `ATM`. If your entity list puts `Account` under `ATM`, you have already made the design decision the case study exists to test.

## Class Design

Start with the bank side, because it is the source of truth and the design falls out of it. The bank's job is to apply a transaction to an account atomically. The critical property is that the check (enough money) and the debit (take the money) happen as one unit. In the interview this is a `synchronized` method or, in prose, "a single atomic database operation." In code, the honest version is a lock around the read-modify-write.

```java
public class Account {
    private final String accountNumber;
    private long balanceInCents;

    public Account(String accountNumber, long balanceInCents) {
        this.accountNumber = accountNumber;
        this.balanceInCents = balanceInCents;
    }

    public synchronized boolean withdraw(long amountInCents) {
        if (balanceInCents < amountInCents) {
            return false;
        }
        balanceInCents -= amountInCents;
        return true;
    }

    public synchronized void deposit(long amountInCents) {
        balanceInCents += amountInCents;
    }

    public synchronized long getBalanceInCents() { return balanceInCents; }
}
```

The `synchronized` on each mutating method is the whole concurrency story in three words. In a real bank this is a database row lock and a transactional update with a `WHERE balance >= amount` guard. Do not write the database here; write the lock, and say out loud that the real system is a row lock inside a transaction.

The bank applies transactions. It returns a `Transaction` object on success so there is a paper trail for reconciliation, which is the non-functional requirement in concrete form.

```java
public class Bank {
    private final Map<String, Account> accounts = new ConcurrentHashMap<>();

    public void addAccount(Account account) { accounts.put(account.getAccountNumber(), account); }

    public Transaction withdraw(String accountNumber, long amountInCents) {
        Account account = accounts.get(accountNumber);
        if (account == null || !account.withdraw(amountInCents)) {
            throw new InsufficientFundsException(accountNumber);
        }
        return Transaction.withdrawal(accountNumber, amountInCents);
    }

    public Transaction transfer(String fromAccount, String toAccount, long amountInCents) {
        // order the two locks to avoid deadlock, see concurrency section
        return null; // elided
    }
}
```

Now the machine side. The `ATMSession` is the authentication gate. Everything else on the machine checks the session before doing anything. A session is born at card insertion, becomes usable when the PIN checks out, and dies when the card is ejected.

```java
public class ATMSession {
    private final String cardNumber;
    private String activeAccountNumber;
    private boolean authenticated;

    public ATMSession(String cardNumber) {
        this.cardNumber = cardNumber;
    }

    public boolean isAuthenticated() { return authenticated; }
    public String getActiveAccountNumber() { return activeAccountNumber; }
    public void bindAccount(String accountNumber) { this.activeAccountNumber = accountNumber; }

    void markAuthenticated() { authenticated = true; }
}
```

The `ATM` ties it together. It holds the session, talks to the bank, and asks the dispenser whether the requested amount is even physically possible before bothering the bank. Order matters here too: check the cassette first, because if the machine cannot hand over the cash, the withdrawal should fail fast and not touch the account.

```java
public class ATM {
    private final Bank bank;
    private final CashDispenser dispenser;
    private ATMSession session;

    public ATM(Bank bank, CashDispenser dispenser) {
        this.bank = bank;
        this.dispenser = dispenser;
    }

    public boolean insertCard(String cardNumber) {
        session = new ATMSession(cardNumber);
        return true;
    }

    public boolean enterPin(String pin) {
        // in a real system this is a call to the bank's auth service
        if (isValidPin(session.getCardNumber(), pin)) {
            session.markAuthenticated();
            return true;
        }
        return false;
    }

    public Transaction withdraw(long amountInCents) {
        if (!session.isAuthenticated()) {
            throw new IllegalStateException("Not authenticated");
        }
        if (!dispenser.canDispense(amountInCents)) {
            throw new IllegalStateException("ATM out of cash or cannot make the amount");
        }
        Transaction txn = bank.withdraw(session.getActiveAccountNumber(), amountInCents);
        dispenser.dispense(amountInCents);
        return txn;
    }
}
```

Diagram: the ownership split that defines the case study, and the two-ATM race the atomic account operation resolves. The balance lives on the bank, never on the ATM.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 445" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ahc" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="920" height="445" fill="#ffffff"/>

  <text x="460" y="26" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">The ATM is a terminal; the bank is the source of truth</text>

  <rect x="20" y="88" width="320" height="190" rx="10" fill="#f8fafc" stroke="#94a3b8" stroke-dasharray="6 4"/>
  <text x="35" y="108" font-size="13" font-weight="bold" fill="#64748b">ATM (terminal)</text>
  <rect x="580" y="88" width="320" height="190" rx="10" fill="#f8fafc" stroke="#94a3b8" stroke-dasharray="6 4"/>
  <text x="595" y="108" font-size="13" font-weight="bold" fill="#64748b">Bank (source of truth)</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ahc)">
    <line x1="260" y1="228" x2="616" y2="172"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12.5" fill="#1f2937">
    <rect x="40" y="120" width="130" height="52" rx="8" fill="#e0e7ff" stroke="#6366f1"/>
    <text x="105" y="142" text-anchor="middle" font-weight="bold" fill="#3730a3">ATMSession</text>
    <text x="105" y="160" text-anchor="middle" font-size="11.5" fill="#4338ca">card → PIN → account</text>
    <rect x="190" y="120" width="130" height="52" rx="8" fill="#e0e7ff" stroke="#6366f1"/>
    <text x="255" y="142" text-anchor="middle" font-weight="bold" fill="#3730a3">CashDispenser</text>
    <text x="255" y="160" text-anchor="middle" font-size="11.5" fill="#4338ca">cassette cash</text>
    <rect x="60" y="200" width="200" height="56" rx="8" fill="#fef3c7" stroke="#f59e0b"/>
    <text x="160" y="223" text-anchor="middle" font-weight="bold" fill="#92400e">ATM.withdraw()</text>
    <text x="160" y="241" text-anchor="middle" font-size="11.5" fill="#b45309">session check → cassette check</text>

    <rect x="620" y="130" width="240" height="80" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="740" y="155" text-anchor="middle" font-weight="bold" fill="#14532d">Account</text>
    <text x="740" y="173" text-anchor="middle" font-size="11.5" fill="#15803d">synchronized withdraw()</text>
    <text x="740" y="191" text-anchor="middle" font-size="11.5" fill="#15803d">balance lives here — never on the ATM</text>
    <rect x="620" y="230" width="240" height="40" rx="8" fill="#f3f4f6" stroke="#cbd5e1"/>
    <text x="740" y="254" text-anchor="middle" font-size="12" fill="#4b5563">Transaction record — for reconciliation</text>
  </g>
  <text x="400" y="236" font-size="12" fill="#475569">withdraw(intent)</text>
  <text x="400" y="252" font-size="12" fill="#475569">atomic debit</text>

  <text x="460" y="318" text-anchor="middle" font-size="15" font-weight="bold" fill="#1f2937">The race: two ATMs, one account, the last $1000</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ahc)">
    <line x1="240" y1="356" x2="536" y2="356"/>
    <line x1="480" y1="372" x2="536" y2="372"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="12.5" fill="#1f2937">
    <rect x="40" y="336" width="200" height="56" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="140" y="359" text-anchor="middle" font-weight="bold" fill="#334155">ATM-1</text>
    <text x="140" y="377" text-anchor="middle" font-size="11.5" fill="#64748b">withdraw(1000)</text>
    <rect x="280" y="336" width="200" height="56" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="380" y="359" text-anchor="middle" font-weight="bold" fill="#334155">ATM-2</text>
    <text x="380" y="377" text-anchor="middle" font-size="11.5" fill="#64748b">withdraw(1000)</text>
    <rect x="540" y="336" width="240" height="56" rx="8" fill="#dcfce7" stroke="#16a34a"/>
    <text x="660" y="359" text-anchor="middle" font-weight="bold" fill="#14532d">Account (balance: 1000)</text>
    <text x="660" y="377" text-anchor="middle" font-size="11.5" fill="#15803d">synchronized withdraw()</text>
  </g>
  <g font-size="12" fill="#475569">
    <text x="660" y="410" text-anchor="middle" fill="#15803d">1st: debits → balance 0</text>
    <text x="660" y="426" text-anchor="middle" fill="#b91c1c">2nd: sees 0 → insufficient funds</text>
  </g>
</svg>
```

Notice the balance never appears on the ATM. To check a balance, the ATM asks the bank. To withdraw, the ATM asks the bank. The machine holds exactly two things: the session and the cash it has not yet handed over. Everything else is a question to the bank.

## Design Patterns Used

The honest pattern answer here is "the one that matters is at the database, not in your class diagram." The single atomic operation on the account is the concurrency pattern that makes the system correct; if the interviewer wants a name, it is a read-modify-write made atomic, which in real systems is a `SELECT ... FOR UPDATE` or an optimistic version check. On the class side, the ATM uses a Facade, in that it presents one object that hides the bank and the dispenser behind it. That is a fine observation and nothing more. There is no Strategy for PIN verification (one mechanism), no Observer for the display (a display can be updated by return values), no Command pattern wrapping each transaction type. The Command pattern is the tempting wrong answer here, because the ATM genuinely does have a menu of operations, but the interview is over before the command hierarchy earns its complexity. Say that directly if pushed.

## Handling Edge Cases / Concurrency

This is the case study's money question, literally. Two ATMs, one account, both try to withdraw the last thousand dollars. Without atomicity, both read a balance of 1000, both pass the check, both debit, and the bank has conjured a thousand dollars. The fix is the lock on `withdraw` inside `Account`: the first caller debits and releases, the second sees 0 and fails. Walk that exact scenario in the interview and you have answered the strongest question in the whole case study.

The second concurrency edge is the transfer, which touches two accounts and therefore two locks. If two transfers run in opposite directions between the same two accounts, naive locking can deadlock. The standard fix is ordering: always acquire the two account locks in the same canonical order, say by account number, and both transfers acquire in that order and cannot cross. This is the detail that separates candidates who have actually thought about concurrency from candidates who read that "synchronized" exists. The dispenser edge cases are physical: an amount the cassette cannot make (withdraw 30 when the machine only has twenties and fifties) must fail before touching the bank, and a dispense failure after a successful debit is a reconciliation problem the ATM logs and the bank can see. Name that ordering: cassette check before the bank call, log after.

## Common Mistakes

The number one mistake is the local balance. `Account` on the ATM with a mutable `balance` field, incremented and decremented by the terminal itself. The interviewer asks "two ATMs, same account, two withdrawals of the full balance" and the design has no answer except to add a lock to the wrong class. The balance belongs to the bank. Full stop.

The second mistake is skipping the cassette check. The design happily debits the account, then discovers the machine cannot dispense the amount, and now has to unwind a transaction after the fact. The check is one line and it belongs before the bank call.

The third mistake is treating every failure as an exception and throwing from the happy path. Insufficient funds is an expected outcome of withdrawing; the ATM should render it as a screen, not as a stack trace. As in the vending machine, result values beat exceptions for expected failures.

## Interview Perspective

A weak answer is a single `ATMSystem` class with methods for everything, or worse, an `ATM` that owns a list of `Account`s. There is no session, no bank, no ownership split, and the follow-up "what if my card is used at two ATMs" ends the conversation.

A strong answer names the split up front: "the ATM is a terminal, the bank is the source of truth, the session gates everything after PIN entry, and the account methods are atomic." The interviewer will then test it. Common twists: "what if the user withdraws then the dispenser jams" (the debit already happened, the bank has a transaction record, reconciliation resolves it; name the log entry), "what if the user takes the card out mid-transaction" (the session dies, the current operation fails atomically, nothing half-applied), "what if the user wants a balance check and the bank is down" (the ATM cannot answer, which is correct behavior, and the honest design says so instead of caching a stale balance). The strongest candidates bring up the two-ATM race themselves and walk through the lock without being prompted.

## Knowledge Check

1. Two ATMs each receive a withdrawal request for the full account balance of 500 at the same instant. Walk through what each `Account.withdraw` call observes and returns, assuming the `synchronized` methods from the design.
2. The ATM checks the cassette after debiting the account. Describe the concrete failure this ordering causes, and where the check belongs instead.
3. A transfer from account A to account B runs concurrently with a transfer from B to A. Explain the deadlock scenario and the lock-ordering rule that prevents it.

## Key Takeaways

- The bank owns the balance. An `Account` with a local mutable balance on the ATM is a failed design.
- A withdrawal is a single atomic read-modify-write. `synchronized` here; a row lock in real life.
- The session gates every operation after PIN entry. No session, no money movement.
- Cassette check before the bank call, reconciliation log after. Ordering is correctness.
- Lock both accounts in canonical order on transfers, or accept the deadlock.
- The ATM's class diagram is boring on purpose. The interesting design is the bank's.

## What's Next

The ATM introduced a client-server split inside a single design, and the lock that makes concurrent money movement correct. The chess game drops the money entirely but keeps the state machine, and adds the requirement that broke the vending machine's scope: a move generator that has to actually play chess.

---

This article explains how to design an ATM around the ownership split where the terminal is dumb and the bank owns balances. Its point of view is that a local balance field fails on arrival, and the atomic account operation is the whole concurrency story.
