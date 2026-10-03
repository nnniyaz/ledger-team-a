# Decision 0001: How data survives a crash

| | |
|---|---|
| Date | 2026-10-02 |
| Status | Accepted by the Team Leader. Team review pending. |
| Decided by | Niyaz Nassyrov |
| Reviewed by | TODO: names and date |

## Context

Ledger must guarantee that an acknowledged write survives a crash at any instant. There are several well-known ways to get that guarantee. They differ in how much code they need, how the correctness argument is structured, and how recovery behaves.

The team is four people who started Ada and SPARK this semester. Three sprints (six weeks) remain after this one.

## Criterion

**Provability within the schedule:** which approach can this team both implement and prove in three sprints?

In practice that breaks down into four questions asked of each option:

1. How small is the invariant that has to be stated and proved?
2. Can the durability argument be isolated in one small module, or is it spread through the engine?
3. How many background or moving parts need their own crash-safety argument?
4. Can four people work on it in parallel?

Raw performance is explicitly not part of the criterion.

## Options considered

### A. Write-ahead log (chosen)

Every change is appended to a log and synced before it is acknowledged. The data file is updated later, at a checkpoint. Recovery replays the committed part of the log.

Specific variant: redo-only, full page images (see design section 4.2).

### B. Shadow paging (copy-on-write tree)

A change never overwrites a page. It writes new copies of the changed pages along the path from leaf to root, then switches a root pointer atomically. No log and no replay.

### C. Append-only log as the store

There is no separate data file. The log is the database: every Put appends a record, an in-memory index maps each key to its latest record, and a compaction step rewrites the log to drop stale records.

## Comparison against the criterion

| Question | A. Write-ahead log | B. Shadow paging | C. Append-only store |
|---|---|---|---|
| Size of the invariant | Small: "set page N to image X" is idempotent | Large: reachability and free-page accounting across the whole tree | Small for writes; a second one for compaction |
| Is durability isolated? | Yes: codec, log and recovery, independent of the index | No: durability is a property of the tree code itself | Yes for writes; no once compaction is included |
| Extra parts with their own crash argument | Checkpoint (reuses the recovery argument) | Free-page reclamation tied to reader snapshots | Compaction and index rebuild |
| Parallel work for four people | Good: log, pager, tree, tests are separate units | Poor: everything meets inside the tree | Medium |
| Recovery cost | Bounded by the log size since the last checkpoint | None | Grows with the total data size (index is rebuilt by a full scan) |

## Decision

Option A, a redo-only write-ahead log with full page images.

## Why the others were rejected

**Shadow paging.** Its real strength is that recovery is trivial and readers get consistent snapshots for free. It loses on the criterion because the crash-safety argument cannot be separated from the tree: to prove it, one has to prove properties of page reachability and free-space management over the whole structure. That is a much larger proof than the team can take on while still learning SPARK, and it cannot be split between people.

**Append-only store.** This was the closest runner-up and is probably the simplest to get working. It was rejected for three reasons. Compaction is a second procedure that must itself be crash-safe, so the simplicity is partly an illusion. Recovery time grows with the total amount of data, not with recent activity, which works against the recovery-speed goal already on the board. And it has no on-disk index, so the B-tree and checkpointing work the team planned in Sprint 1 would have no place.

## Consequences

- Each small write puts at least one full page (4096 bytes) in the log. Write volume is high. Accepted for a prototype.
- A checkpoint is required to bound the log and the recovery time.
- The durability proof is confined to `Wal_Codec` and `Recovery`.

## When to revisit

If the B+tree is not working by the end of Sprint 4, fall back to option C for the index side: keep the write-ahead log and the proofs, and replace the tree with an in-memory index rebuilt on open. The durability work would carry over unchanged.
