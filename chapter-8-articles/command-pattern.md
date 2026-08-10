# Command Pattern

## Learning Objectives

- Package an action as an object, so a method call, its receiver, and its arguments travel together and can be stored, queued, and undone.
- Build the invoker-receiver split: the invoker only knows how to run the command, not what it does, and the receiver carries the real work.
- Explain what commands buy beyond a method call, queuing, undo, retry, and logging, and when those buys do not apply.

## Introduction

Command turns a request into a standalone object. Instead of a caller invoking `receiver.doSomething(arg)`, the caller constructs a command object holding the receiver and the argument, hands it to an invoker, and the invoker calls `command.execute()` later. The request becomes data, and being data is the whole point: data can be queued, stored, retried, and undone in ways a direct method call cannot.

The pattern separates the decision to act, the act itself, and the code that performs the act. The one who wants the thing done, the invoker, does not understand the thing. The thing is packaged as a command.

## Problem Statement

Here is a system where the request is welded to its timing. A text editor's buttons should act immediately, but a macro should replay the same actions later, and undo should reverse them. If the editor's button handler calls `editor.deleteWord()` directly, then replaying is impossible, because there is no stored record of what was deleted or in what order, and undo has nothing to walk back. The button, the macro, and the undo stack all need the same things, the exact actions and their order, but only one of them, the button, can currently happen at all.

The same shape appears outside editors. A monitoring alarm wants a job re-run when it retries. An audit log wants each user action recorded with its parameters. A job queue wants work submitted now and executed on another thread. In every case the failure is that the action exists only at the moment it is called. The moment the call returns, the action is gone, and nothing can re-run, undo, or log it. The action needs a body that outlives the call.

## Core Concept

Command gives the action a body. One interface names the operation:

```java
public interface Command {
    void execute();
}
```

An action becomes a concrete command that captures its receiver and arguments:

```java
public class InsertTextCommand implements Command {
    private final Editor editor;
    private final String text;
    private final int position;

    public InsertTextCommand(Editor editor, String text, int position) {
        this.editor = editor;
        this.text = text;
        this.position = position;
    }

    @Override
    public void execute() {
        editor.insertInternalAt(position, text);
    }
}
```

The receiver does the real work through methods the command calls. The invoker knows only the `Command` interface:

```java
public class Button {
    private Command action;

    public void setAction(Command action) {
        this.action = action;
    }

    public void onClick() {
        action.execute();
    }
}
```

The wiring binds a button to an action:

```java
Button saveButton = new Button();
saveButton.setAction(new SaveCommand(document, fileStore));
```

Nothing here is deep, but the indirection has already bought something. The button does not know what saving means. The command does. A queue can now hold commands:

```java
public class CommandQueue {
    private final Deque<Command> commands = new ArrayDeque<>();
    private final ExecutorService executor = Executors.newFixedThreadPool(4);

    public void enqueue(Command command) {
        executor.submit(() -> { command.execute(); });
    }
}
```

And a macro can be a composite of commands, which is just a list followed in order, and undo, which stores an inverse or a snapshot, becomes possible because the actions exist as objects. The request stopped living and dying at the call site. That is the entire payoff.

Diagram: command, invoker, and receiver

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1000 400" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="160" width="240" height="90" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="130" y="186" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Invoker</text>
  <text x="130" y="212" text-anchor="middle" font-size="12" fill="#1a2733">-action: Command</text>
  <text x="130" y="236" text-anchor="middle" font-size="12" fill="#1a2733">+onClick()</text>

  <rect x="420" y="60" width="160" height="70" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="500" y="82" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="500" y="104" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Command</text>
  <text x="500" y="128" text-anchor="middle" font-size="12" fill="#1a2733">+execute()</text>

  <rect x="420" y="270" width="280" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="500" y="292" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">InsertTextCommand</text>
  <text x="500" y="318" text-anchor="middle" font-size="12" fill="#1a2733">+execute()</text>

  <rect x="760" y="60" width="200" height="240" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="860" y="120" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Receiver</text>
  <text x="860" y="150" text-anchor="middle" font-size="12" fill="#1a2733">+insertInternalAt()</text>
  <text x="860" y="180" text-anchor="middle" font-size="12" fill="#1a2733">+deleteWord()</text>
  <text x="860" y="210" text-anchor="middle" font-size="12" fill="#1a2733">+saveDocument()</text>

  <line x1="220" y1="185" x2="418" y2="125" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="500" y1="210" x2="500" y2="132" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="580" y1="255" x2="758" y2="180" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

The invoker calls the command, the command calls the receiver, and the receiver does the work. The invoker never touches the receiver, and the command is the only thing that knows the receiver exists. That is the split the pattern is really selling.

### What the pattern genuinely buys

The honest list of why you would reach for commands:

- Queuing. Actions become values a queue can hold, which is the entire `Executor` framework.
- Replay and macro. A list of commands is a script, and re-running it reproduces the sequence.
- Undo and redo. With the command and its inverse, `undo()` mirrors `execute()`, and a stack of them walks time both ways.
- Composability. A macro command is a list of commands, and no command knows it is part of a list.
- Auditing. Recording the command and its parameters captures the action even after it ran.

The honest counterpoint: a command is a class, and classes cost. For one-off local calls, `receiver.doSomething()` is the better code, and turning it into a command to feel architectural is overhead. The pattern pays when the action truly needs a life beyond its call site, queued or stored or reversed. When it does not, this is the wrong shape.

## Real Production Usage

The `Executor` framework is the command pattern at its most literal. Every `Runnable` and `Callable` you submit to an `ExecutorService` is a command, and `submit` is the invoker that hands it to a worker; the framework runs it later, maybe on another thread, and returns a `Future` for the result. The command abstraction is what makes thread pooling work, because a thread has no idea what task it is running.

AWT and Swing use commands for the same reason they use observers: decoupling. A menu item and a button can share one `Action`, and an `AbstractAction` is a command with attached state, so the same action object updates both trigger and enabled-ness. Spring Batch builds entire jobs from chunks and steps. That is command composition at enterprise scale: a `Job` is a list of steps, each step is named a command. Sending an email, committing a transaction, and issuing a CLI shell are all modeled as a `Runnable` in various frameworks, and let the command object decouple the request from the timing. When you see a framework that lets you "schedule this to run later," you are looking at commands behind it.

## Common Mistakes

**Stuffing logic into execute().** If the command implements the whole behavior, it is a class wearing a method. The command should capture the request and delegate the real work to a receiver, so the behavior stays in the domain where it is unit-testable and reusable. A command full of `if` and business rules is a strategy in disguise.

**Putting the invoker inside something heavy.** If the invoker, the button or the controller, computes results and business rules before calling `execute()`, the decision and the action have coupled again. The invoker should be thin and dumb.

**Turning every call into a command "for flexibility that never materialized."** The cost of a command is real: a class per action and an indirection layer. If nothing will ever queue or undo the action, the command is dead weight. Use the pattern only where life the command outside the call site is a concrete requirement.

## Interview Perspective

Command is where interviewers check whether you grasp what data-izing an action unlocks. A weak answer draws the command class and the invoker. A strong answer says the point is that the action outlives its invocation, and can extend, "so a queue can hold it, a thread pool can run it, an undo can reverse it, and a macro can repeat it." The skill is naming the buys, not the drawing.

The follow-up that sorts people is the undo question. "How do you undo a command?" A weak answer forgets that `execute()` has to capture enough state to reverse. A strong answer explains that some commands carry an `undo()` holding a snapshot, and some are irreversible. Inserting text is undoable only if the command captured the removed text; a bank transfer settled elsewhere is not cleanly reversible, and you may have to design the receiver to support compensation. Undo is a second contract on the command, and it might shape the receiver from the start.

Common follow-ups:

- "How does undo work when the command mutates an external system, like a bank transfer?"
- "What is the difference between Command and Strategy?"

## Knowledge Check

1. Add an `undo()` method to `InsertTextCommand`. What state does the command have to capture, and when is the capture made?
2. A macro is a list of commands. Name one property of the composite command that an individual command cannot give, and one it cannot.
3. A REST handler immediately executes a request. Justify whether turning the request into a command pays for itself here, and when it would.

## Key Takeaways

- Command packages an action as an object, so the operation can be queued, stored, retried, and reversed.
- The invoker only knows to `execute()`; the command names the receiver; the receiver does the work.
- Queuing, macros, undo, and retries are what the pattern buys, and each is optional but worthwhile.
- The `Executor` framework and Swing's shared actions are Command in production.
- A command is only worth its class when the action needs a life beyond the call site; otherwise it is ceremony.

## What's Next

The next article is Chain of Responsibility, which puts a sequence of handlers in front of the request. A request travels down an ordered list of handlers until one claims it, and each handler gets the chance to act, pass on, or stop. We will cover the short-circuit, the ordering, and the single rule that keeps a chain from becoming a tangle.

---

This article explains the Command pattern as packaging an action into an object so it can be queued, replayed, and undone. It argues that the pattern pays off exactly when the action needs a life beyond its call site.