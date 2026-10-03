# Decision 0003: What is proved in SPARK

| | |
|---|---|
| Date | 2026-10-02 |
| Status | Accepted by the Team Leader. Team review pending. |
| Decided by | Niyaz Nassyrov |
| Reviewed by | TODO: names and date |

## Context

The project title promises an engine "you can prove correct". SPARK lets a team choose how much to prove. AdaCore describes the choice as assurance levels:

| Level | What is established |
|---|---|
| Stone | The code is valid SPARK |
| Bronze | Correct initialization and data flow |
| Silver | Absence of run-time errors |
| Gold | Proof of selected key properties |
| Platinum | Full functional correctness |

Each level costs noticeably more than the one before. The team has to decide where to spend a limited proof budget.

## Criterion

Same as decisions 0001 and 0002: **provability within the schedule.** Here it reads: which proof target can a team that is still learning SPARK actually reach in three sprints, while still backing the claim in the project title with proof and not only with tests?

## Options considered

### A. Silver only

Prove absence of run-time errors in all SPARK code. Check all behaviour with tests.

### B. Silver everywhere, Gold for the durability core (chosen)

Silver for all SPARK code. Gold for two properties:

1. **WAL codec round trip.** Decoding an encoded record returns the same record, and a buffer with a wrong checksum is rejected.
2. **Recovery idempotence.** Over a model of the data file, applying the committed log twice gives the same result as applying it once.

Behaviour below the I/O boundary is covered by crash tests with fault injection.

### C. Gold for the write-ahead log and the B+tree

Additionally prove functional properties of the tree: after an insert, a lookup of that key returns the inserted value; the tree stays sorted and balanced.

## Comparison against the criterion

| Question | A. Silver only | B. Silver + Gold core | C. Gold for log and tree |
|---|---|---|---|
| Reachable in three sprints by this team | Yes | Likely: the two properties are small and self-contained | Unlikely: tree proofs need ghost models and lemmas the team has not written before |
| Backs "prove correct" with proof | Weakly: shows no crashes in the code, says nothing about data | Yes, for the part of the claim that matters most: recovery | Yes, more broadly |
| Risk if the proof stalls | Low | Contained: the Gold units are separate from the rest | High: tree proof and tree implementation block each other |

## Decision

Option B.

## Why the others were rejected

**Silver only.** Safe for the schedule, but it does not answer the project's own question. Silver shows the code does not raise an error; it does not show that data comes back after a crash. A project named for provable crash safety should prove something about crash safety.

**Gold for the log and the tree.** The strongest result, and the wrong risk for this team. Functional proofs of a B+tree need a ghost model of the tree as a map, loop invariants over splits, and lemmas connecting the two. Committing to that while still learning the tool would put the whole engine at risk if the proofs stall.

## What is not proved, and what covers it

| Not proved | Covered by |
|---|---|
| The operating system and the disk behave as `Ledger.IO` assumes | Crash tests with fault injection (design section 9.2) |
| B+tree functional behaviour | Unit tests; Silver still guarantees no run-time errors |
| Behaviour with concurrent readers | Tests in Sprint 5 |

## Consequences

- `Wal_Codec` must be pure functions over bounded byte arrays so the round trip can be stated as a postcondition.
- `Recovery` needs a small ghost model of the data file (an array of pages) to state idempotence.
- GNATprove runs in CI from Sprint 3 so that Silver is kept as code is added, not recovered at the end.

## When to revisit

- If recovery idempotence is not proved by the end of Sprint 4, the target for that unit drops to Silver plus crash tests, and this record is updated to say so.
- If both Gold proofs are done by the middle of Sprint 4, the stretch goal is the sortedness invariant of a single B+tree node under insert.
