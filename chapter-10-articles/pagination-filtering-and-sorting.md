# Pagination, Filtering, and Sorting

## Learning Objectives

- Choose between offset and cursor pagination, and name the consistency each one gives up or protects.
- Describe a paginated response so a client can walk the entire set with nothing to guess.
- Apply filtering and sorting before the window, so every page belongs to the same view of the data.

## Introduction

A collection endpoint that returns everything is not a collection, it is a dump. It grows with the database until one request is the whole table serialized. Pagination is how you bound each response and still let a client reach all of it in order. Filtering and sorting are the two knobs that make a page mean something specific instead of a wad of shared rows.

## Problem Statement

The first failure is the whole-tables response. `GET /orders` returns every order in one body, the JSON size grows with the table, the first call is a slow and memory-hungry monster, and the client's memory climbs with it. Add pagination and the response is bounded at last, but the walk still has to happen without the client guessing.

The next failure is the page you cannot trust. A `page` and `size` that do not return a cursor or a total, and the client has no way to know when it is done, or which rows it has seen. And the offset page that lies as new rows, returning the same item twice and another not at all, breaks the last loop that thought it was walking a stable list.

## Core Concept

### The three controls

A paged endpoint has three controls: the filter, the sort, and the page. The filter narrows the set before anything else, the sort fixes the order the set comes out, and the page selects the window on the narrowed, ordered set.

```java
Map<String, String> controls = Map.of(
    "status",  "active",
    "sort",    "createdAt,desc",
    "limit",   "20"
);
```

The three are orthogonal: change the filter, you look at a different set; change the sort, same set in a different order; move the page, same set and order, deeper window. If you pin the sort and let the filter shift, the pages the client walks are no longer part of one stable list.

### Offset pagination: the cheap default

The simplest form is offset. `page=2&per_page=20`, the server counts rows and skips the first forty, and it is instant to write and to understand. But under writes it breaks. A row inserted in the middle of the data shifts everything after it by one, so your second page skips a row and your later page repeats one. The set is no longer stable, and the client that walks it gets a mutated list.

And the offset cost swaps with the table size. An `OFFSET 100000` must count and discard a hundred thousand rows to return twenty, which is a slow scan every page. Fine for a small, static table; wrong for the one that grows and changes.

### The cursor keeps the page on track

Cursor pagination returns a value instead of a count, and the next page begins where that value points. The server anchors a keyset, the last row's `id` (or timestamp), and the next page is `WHERE id > cursor ORDER BY id LIMIT 20`.

```java
@Query(value = """
    SELECT o FROM Order o
    WHERE o.id > :cursor
    ORDER BY o.id
    LIMIT :limit
    """)
List<Order> findPage(@Param("cursor") Long cursor, @Param("limit") int limit);
```

The row that arrives in the middle does not shift the pages, because the next page does not begin at a count, it begins at the anchor. The client follows the `nextCursor` returned, and each row arrives exactly once.

The sharp rule with a cursor: it has to ride on a stable column. Order the anchor with at least `id` as a tiebreaker, or two rows with the same timestamp sit in unsortable territory. The decision tree is the table: small and mostly static, offset is fine; large or written to, the cursor is the one that holds.

### The paginated response

The response carries the rows and the navigation in one envelope:

```java
public record Page<T>(List<T> data, String nextCursor, boolean hasMore) {}
```

`data` is the slice. `nextCursor` is the value the caller sends back as the `cursor`. `hasMore` is the flag that turns the walk into a real loop: if there are fewer than `pageSize` rows for the page, no more, and the client stops. The client cannot know `hasMore` from the length alone, so the flag is what makes the loop finite.

Two details make the envelope survive contact with clients. First, cap the page size. A client can ask for ten thousand rows and the server is within its rights to answer with an error or to clamp to a documented maximum, and the contract should say which. Unbounded `per_page` is just the whole-table dump in smaller increments. Second, keep the envelope shape constant across every collection endpoint. When the paging fields are the same names in the same place everywhere, a client writes the walk loop once and reuses it, which is the consistency this chapter keeps coming back to.

Diagram: offset counts items, cursor anchors at a value

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 440" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <rect x="40" y="50" width="380" height="350" rx="12" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <rect x="70" y="80" width="150" height="40" rx="6" fill="#eef2f7" stroke="#9aa7b4" stroke-width="1"/>
  <text x="145" y="106" text-anchor="middle" font-size="11" fill="#5a6b7a">item</text>
  <rect x="70" y="136" width="150" height="40" rx="6" fill="#eef2f7" stroke="#9aa7b4" stroke-width="1"/>
  <text x="145" y="162" text-anchor="middle" font-size="11" fill="#5a6b7a">item</text>
  <rect x="70" y="192" width="150" height="40" rx="6" fill="#eef2f7" stroke="#9aa7b4" stroke-width="1"/>
  <text x="145" y="218" text-anchor="middle" font-size="11" fill="#5a6b7a">item</text>
  <rect x="70" y="248" width="150" height="40" rx="6" fill="#fdf0ef" stroke="#a94442" stroke-width="1.5"/>
  <text x="145" y="274" text-anchor="middle" font-size="11" fill="#a94442">page 2</text>
  <text x="145" y="316" text-anchor="middle" font-size="11" fill="#5a6b7a">counted from row one</text>

  <rect x="500" y="50" width="380" height="290" rx="12" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <rect x="530" y="80" width="150" height="40" rx="6" fill="#eef2f7" stroke="#9aa7b4" stroke-width="1"/>
  <text x="605" y="106" text-anchor="middle" font-size="11" fill="#5a6b7a">item</text>
  <rect x="530" y="136" width="150" height="40" rx="6" fill="#eef2f7" stroke="#9aa7b4" stroke-width="1"/>
  <text x="605" y="162" text-anchor="middle" font-size="11" fill="#5a6b7a">item</text>
  <rect x="530" y="192" width="150" height="40" rx="6" fill="#d6e6f7" stroke="#2a6fbf" stroke-width="1.5"/>
  <text x="605" y="218" text-anchor="middle" font-size="11" fill="#2a6fbf">anchor id</text>
  <rect x="530" y="248" width="150" height="40" rx="6" fill="#eef2f7" stroke="#9aa7b4" stroke-width="1"/>
  <text x="605" y="274" text-anchor="middle" font-size="11" fill="#5a6b7a">item</text>
  <text x="605" y="316" text-anchor="middle" font-size="11" fill="#5a6b7a">page starts after id</text>

  <text x="255" y="30" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">OFFSET</text>
  <text x="720" y="30" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">CURSOR</text>
</svg>
```

Left, the page is rows counted from row one, so a new first row shifts the window. Right, the page starts after a fixed anchor, the `id`, so a new row does not move it. The anchor, not a count, is what makes "page 2" stable.

### Filter and sort ahead of the window

The slicing happens on the filtered and sorted set, or the page is meaningless. A request with `status=active` and `page 2` must be page two of the active rows, and a `sort=createdAt,desc` must order the whole filter before the window cuts. The three run as: filter, sort, window. Swap the order and the page has rows that do not belong to the same story, which is a client that sees a mismatched list. Apply filter and sort before the offset or the cursor, whichever paginator you use.

## Real Production Usage

The high-churn systems move the same way every time: the timelines, the audit trails, the live feeds, all switch to cursor-based walking the day the table outgrows the offset. Spring Data defaults to `Pageable`, `page` and `size`, which maps to an `OFFSET`, and it gives you `Page` and `hasNext()` for free; that is the right pick for small, mostly-static collections. The upgrade happens in the query: the hot repository swaps the find method to a keyset query on `id`, and the pages stop drifting.

The rule of thumb that holds across teams: offset when the table is small and calm, cursor when it is large or written to. The reason is mechanical. An offset on a large table costs a scan and drifts under writes, while a cursor costs one index lookup and is stable. The teams that mispredict get page skips in production.

## Common Mistakes

**Offset on a hot collection.** Counting rows under writes shifts the pages. Use the cursor on live data, the offset for the table that does not change.

**The page with no way forward.** A response without `nextCursor` or `hasMore` is a page with no path; the client cannot finish the walk. Return the cursor and the flag.

**Window before the filter.** Slicing the raw table, then filtering, then a page mixed of the wrong rows. Filter and sort ahead of the window, always.

## Interview Perspective

The pagination question tests whether you pick the consistency model. A weak answer says "pass a page number." The strong answer names the two approaches and the trade: the offset is simple and cheap, it drifts under writes and costs a scan; the cursor is stable and costs more, but it is the one to use where the table changes. The envelope matters too: data, nextCursor, hasMore.

The one follow-up that settles it is "you page a table that keeps getting new rows mid-walk. Why does offset break?" The candidate who names the shifted window and the repeated rows is all the way through. The candidate who defends offset with "it's simpler" has not hit a live growing set yet.

Common follow-ups:

- "How does an offset page start to repeat rows under concurrent inserts?"
- "You are sorting by time. Why does the cursor need an id tiebreaker?"

## Knowledge Check

1. Orders arrive while a client walks by offset. What does the set page do, and why?
2. Explain why the cursor has to be a stable column and an id tiebreaker, what gives when it is not.
3. A request has `status=suspended`, `sort=createdAt,desc` and `page=2`. What order do filter, sort, and window run in for it to be true page two of the filter?

## Key Takeaways

- Offset is simple and drifts under writes and slow on large tables.
- A cursor anchors the page at a value, so insertions do not move it.
- The cursor rides a stable column with an id tiebreaker.
- Filter and sort run ahead of the window, never after.

## What's Next

A client can walk a paged set; the next article is about the set changing out from under the contract. API versioning strategies covers what happens when the shape you shipped cannot be kept: the choices of URL versus header, how to run two shapes at once, and how to retire an old version without breaking the caller who still points at it.

---

This article explains pagination as a bounded walk, the cursor when the set changes and the offset when it is static. It argues an offset on a live table drifts, and the cursor on a stable key holds.