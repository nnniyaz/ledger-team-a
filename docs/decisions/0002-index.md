# Decision 0002: Index structure

| | |
|---|---|
| Date | 2026-10-02 |
| Status | Accepted by the Team Leader. Team review pending. |
| Decided by | Niyaz Nassyrov |
| Reviewed by | TODO: names and date |

## Context

The engine needs a structure that finds the value for a key. The choice determines the on-disk layout, how much of the engine runs in the background, and which invariants have to be proved.

## Criterion

Same as decision 0001: **provability within the schedule.** For an index that means:

1. Is the structural invariant local (checkable one node at a time) or global?
2. Does it need background work with its own crash-safety argument?
3. Does it fit the paged write-ahead log chosen in decision 0001 without extra machinery?
4. Is it bounded, so that it can be written without heap allocation?

## Options considered

### A. B+tree (chosen)

Fixed-size pages. Internal nodes hold separator keys and child page numbers; leaves hold keys and values in sorted order.

### B. LSM-tree

Writes go to an in-memory table that is flushed to sorted, immutable files. Background compaction merges those files.

### C. Hash index in memory

A hash table maps each key to the position of its record. The table lives in memory and is rebuilt when the database is opened.

## Comparison against the criterion

| Question | A. B+tree | B. LSM-tree | C. Hash index |
|---|---|---|---|
| Invariant | Local: keys sorted within a node, count within bounds, separators bracket children | Global: ordering and visibility across levels and files | Local, simple |
| Background work | None | Compaction, with its own crash-safety argument | None, but needs a full rebuild on open |
| Fit with the paged log | Direct: a node is a page, a change is a set of page images | Poor: it has its own files and its own recovery | Not paged; needs a different storage layout |
| Bounded, no heap | Yes: a node is a fixed array | Hard: memory table and file sets vary in size | Only if the whole key set is bounded by memory |
| Ordered access | Yes | Yes | No |

## Decision

Option A, a B+tree with fixed-size slots.

## Why the others were rejected

**LSM-tree.** It is the better structure for write-heavy workloads, and that is not what this project is judged on. Compaction is a concurrent background process that rewrites files; proving, or even reliably crash-testing, the interaction between compaction, flush and recovery is well beyond what the team can do in three sprints.

**Hash index in memory.** The easiest of the three to implement. It was rejected because the whole index must fit in memory and must be rebuilt by scanning all data on every open, so recovery time grows with the size of the database. It also gives no ordered access, which closes the door on range scans if the Product Owner asks for them.

## Consequences

- Fixed-size slots waste space for short keys and values. Accepted.
- Insert needs page splits, the hardest part of the tree to get right. It gets the most unit tests.
- Delete does not rebalance (design section 6.3). This keeps the proof burden down and is flagged to the Product Owner.

## When to revisit

If the tree is not working by the end of Sprint 4, option C becomes the fallback, as described in decision 0001.
