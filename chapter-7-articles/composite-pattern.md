# Composite Pattern

## Learning Objectives

- Compose objects into tree structures and explain why the client must be unable to tell a leaf from a branch.
- Implement the uniform `Component` contract in Java and see the recursion it enables in methods like `size()`.
- Argue where the uniformity should stop, which is the composite's real design pressure.

## Introduction

Composite lets you treat individual objects and compositions of objects uniformly. A directory and a file are different things, but the code that walks them should not have to care which one it is holding. The pattern makes that true by putting both behind one interface and letting containers hold children of that same interface.

The power is recursive. A directory's size is the sum of its children's sizes, each child a file or another directory. Because both are `FileSystemNode`, the directory can ask each child for its size without knowing what it is, and the child, if it is a directory, does the same thing one level down. The uniformity is not a convenience. It is the mechanism.

## Problem Statement

Here is the code that forces you to check types. A file system renderer wants to print a tree:

```java
public void printNode(FileSystemNode node, String indent) {
    if (node instanceof Directory dir) {
        System.out.println(indent + dir.getName() + "/");
        for (FileSystemNode child : dir.children()) {
            printNode(child, indent + "  ");
        }
    } else if (node instanceof File file) {
        System.out.println(indent + file.getName() + " (" + file.size() + " bytes)");
    }
}
```

It works, and it is exactly the shape that decays. Every piece of code that walks the tree now contains this `instanceof` fork. The renderer, the search, the size report, the copy operation, they all reimplement the same if-else. Add a third node type, say a symlink, and every one of those walkers grows another branch. The `Directory` case is the worst offender: it knows how to recurse, and it embeds that knowledge in every consumer, so the consumers all have to know the container's internals.

The failure is that the tree structure, which is the domain's whole shape, is duplicated across every algorithm that touches it. There is exactly one place that should know how to recurse, and it is the directory itself.

## Core Concept

Composite puts the recursive contract on the nodes. One interface covers both kinds, and the container's methods recurse naturally:

```java
public interface FileSystemNode {
    String getName();
    long size();
    List<FileSystemNode> children();
    void add(FileSystemNode child);
    void remove(FileSystemNode child);
}
```

The leaf implements the interface with no children, and refuses the container operations:

```java
public class File implements FileSystemNode {
    private final String name;
    private final long bytes;

    public File(String name, long bytes) {
        this.name = name;
        this.bytes = bytes;
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public long size() {
        return bytes;
    }

    @Override
    public List<FileSystemNode> children() {
        return List.of();
    }

    @Override
    public void add(FileSystemNode child) {
        throw new UnsupportedOperationException("a file has no children");
    }

    @Override
    public void remove(FileSystemNode child) {
        throw new UnsupportedOperationException("a file has no children");
    }
}
```

The branch holds children and delegates the work down the tree:

```java
public class Directory implements FileSystemNode {
    private final String name;
    private final List<FileSystemNode> children = new ArrayList<>();

    public Directory(String name) {
        this.name = name;
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public long size() {
        long total = 0;
        for (FileSystemNode child : children) {
            total += child.size();
        }
        return total;
    }

    @Override
    public List<FileSystemNode> children() {
        return Collections.unmodifiableList(children);
    }

    @Override
    public void add(FileSystemNode child) {
        children.add(child);
    }

    @Override
    public void remove(FileSystemNode child) {
        children.remove(child);
    }
}
```

Now the renderer loses its fork:

```java
public void printNode(FileSystemNode node, String indent) {
    System.out.println(indent + node.getName() + " (" + node.size() + " bytes)");
    for (FileSystemNode child : node.children()) {
        printNode(child, indent + "  ");
    }
}
```

The leaf returns an empty list from `children()`, so the renderer recurses into nothing and stops. That is the trade the pattern asks you to accept: the interface carries a child accessor that half its implementors answer with an empty list. The payoff is that one definition of `printNode` handles the whole tree, and no consumer ever forks on node type again. The `size()` method is the same story: one definition, recursion for free.

Diagram: composite tree structure

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1080 700" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="30" width="180" height="60" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="130" y="52" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Client</text>
  <text x="130" y="76" text-anchor="middle" font-size="12" fill="#1a2733">+operation()</text>

  <rect x="420" y="30" width="240" height="80" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="540" y="48" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="540" y="68" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">FileSystemNode</text>
  <text x="540" y="90" text-anchor="middle" font-size="12" fill="#1a2733">+size() +add(child)</text>

  <line x1="220" y1="60" x2="418" y2="60" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>

  <rect x="420" y="180" width="240" height="90" fill="#e9f5ee" stroke="#2e7d4f" stroke-width="1.5"/>
  <text x="540" y="202" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Directory</text>
  <text x="540" y="228" text-anchor="middle" font-size="12" fill="#1a2733">-children: List</text>
  <text x="540" y="252" text-anchor="middle" font-size="12" fill="#1a2733">+size() +add(child)</text>

  <line x1="540" y1="180" x2="540" y2="112" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>

  <line x1="540" y1="270" x2="540" y2="320" stroke="#2e7d4f" stroke-width="1.5"/>
  <line x1="160" y1="320" x2="920" y2="320" stroke="#2e7d4f" stroke-width="1.5"/>
  <line x1="160" y1="320" x2="160" y2="378" stroke="#2e7d4f" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="540" y1="320" x2="540" y2="378" stroke="#2e7d4f" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="920" y1="320" x2="920" y2="378" stroke="#2e7d4f" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="60" y="380" width="200" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="160" y="402" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">File</text>
  <text x="160" y="428" text-anchor="middle" font-size="12" fill="#1a2733">+size()</text>

  <rect x="420" y="380" width="240" height="90" fill="#e9f5ee" stroke="#2e7d4f" stroke-width="1.5"/>
  <text x="540" y="402" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Directory</text>
  <text x="540" y="428" text-anchor="middle" font-size="12" fill="#1a2733">-children: List</text>
  <text x="540" y="452" text-anchor="middle" font-size="12" fill="#1a2733">+size() +add(child)</text>

  <rect x="820" y="380" width="200" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="920" y="402" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">File</text>
  <text x="920" y="428" text-anchor="middle" font-size="12" fill="#1a2733">+size()</text>

  <line x1="540" y1="470" x2="540" y2="540" stroke="#2e7d4f" stroke-width="1.5"/>
  <line x1="520" y1="540" x2="740" y2="540" stroke="#2e7d4f" stroke-width="1.5"/>
  <line x1="520" y1="540" x2="520" y2="578" stroke="#2e7d4f" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="740" y1="540" x2="740" y2="578" stroke="#2e7d4f" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="420" y="580" width="200" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="520" y="602" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">File</text>
  <text x="520" y="628" text-anchor="middle" font-size="12" fill="#1a2733">+size()</text>

  <rect x="640" y="580" width="200" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="740" y="602" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">File</text>
  <text x="740" y="628" text-anchor="middle" font-size="12" fill="#1a2733">+size()</text>
</svg>
```

The tree shows the pattern's shape: directories hold other nodes, files hold nothing, and both answer `size()` the same way. The recursion lives in the directory, and the client never sees a fork.

### The design pressure: how uniform is uniform

There is a famous trade buried in the pattern. The GoF puts `add`, `remove`, and child access on the `Component` interface, which is why the `File` above has to throw `UnsupportedOperationException`. That exception is the smell. It means the interface is too big for half its implementors, and any code that calls `add` on a node it cannot prove is a directory will blow up at runtime.

Two ways out. The "safety first" approach drops the child operations from the interface and gives them only to the composite, which costs you the uniform recursion on children. The "uniformity first" approach keeps them and documents the contract. Real libraries pick both sides. `java.awt.Container` keeps `add` only on containers, which is the safety-first version, and code that wants to add a child to a `Component` has to check. The DOM, by contrast, keeps `appendChild` on every node and defines the leaf behavior as an exception, the uniformity-first version. Neither is wrong. What is wrong is pretending the tension does not exist, so pick one and make the failure mode loud. An `UnsupportedOperationException` with a clear message is loud. A method that silently does nothing is not.

## Real Production Usage

`java.awt.Component` and `java.awt.Container` are the canonical composite: a `Container` holds `Component` children, a panel can contain buttons and other panels, and drawing, layout, and repainting all treat the hierarchy uniformly. Swing's `JComponent` extends the same idea, which is why a `JPanel` inside a `JPanel` just works. The DOM is the other canonical case: `Node` is the component, `Element` is the composite, and text nodes are the leaves. JavaFX's `Region` and `Pane` repeat the shape. When you see a UI toolkit, you are almost certainly looking at Composite holding it together.

## Common Mistakes

**Making the interface so uniform it throws on half its callers.** If `add` exists on the interface, every call site has to assume it can fail. Either give the composite its own child operations, or commit to the uniform contract and make the leaf failure loud and documented. Half measures, silent no-ops, are how composite bugs hide.

**Expecting the tree to be shallow.** The pattern recurses by design, and deep trees mean deep call stacks. A naive `size()` on a million-file tree is a recursive walk with an `ArrayList` per directory. The pattern is not a license to ignore traversal cost; you still own the algorithm.

**Using the pattern for flat structures.** Composite is for genuine part-whole hierarchies. A `List` of homogeneous items is not a composite; forcing the pattern onto it adds an interface and recursion where a loop would do.

## Interview Perspective

Composite is a pattern interviewers use to check two things: whether you understand recursive structures, and whether you understand the uniformity trade, which is the part most people never touch. A weak answer draws a tree and says "leaves and composites implement the same interface." A strong answer draws the tree, shows the recursion in `size()`, and can argue the `UnsupportedOperationException` trade from both sides.

The follow-up is usually about the tension. "Your interface has `add` on it, and a leaf cannot add. Is that a design flaw?" The strong answer names both options and picks one with a reason, rather than insisting the textbook is right.

Common follow-ups:

- "What breaks if the composite's `add` accepts a node that is its own ancestor?"
- "Should child operations live on the interface or only on the composite? Defend it."

## Knowledge Check

1. Trace `directory.size()` on the tree in the diagram above and show where recursion terminates for a file versus a directory.
2. A colleague wants to remove `add` and `remove` from `FileSystemNode` to avoid the throwing `File`. What does the client code that builds the tree now have to do, and what uniformity do you lose?
3. `java.awt.Container` keeps `add` off the `Component` interface. Describe the shape of the code a caller writes to add a button to a panel, and compare its safety with the DOM's approach.

## Key Takeaways

- Composite makes leaves and branches interchangeable behind one interface, which is what makes tree algorithms recursive instead of forked.
- The container recurses by asking each child to do the work, and the child answers the same question whether it is a file or another directory.
- The uniformity trade, `add` on the interface versus on the composite, is the pattern's real design decision and it has two defensible answers.
- UI toolkits, awt, Swing, JavaFX, and the DOM are Composite in daily use.
- The pattern does not make deep trees cheap; you still own the traversal cost.

## What's Next

The next article is Decorator, which looks like Composite's cousin and is really its opposite. Composite builds trees by nesting. Decorator builds a stack by wrapping, each layer adding one responsibility, and the classic example is the Java I/O stack where a stream gets buffered, then checked, then counted. We will cover the wrapping mechanics and the identity problems that wrapping introduces.

---

This article explains the Composite pattern as a uniform interface that lets a client treat a single file and a whole directory tree identically, with recursion living inside the container. It argues that the real design decision is how uniform the interface should be, and that the leaf's throwing methods are a deliberate trade, not a flaw.
