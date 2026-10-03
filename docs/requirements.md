# Ledger: Requirements

| | |
|---|---|
| Project | Ledger, a crash-safe storage engine you can prove correct |
| Team | Team A, FIU Capstone 2, Fall 2026 |
| Product Owner | Olivier Henley (AdaCore) |
| Version | 0.1 draft, 2026-10-02 |
| Status | Written by the Team Leader. Pending review by the team and the Product Owner. |

## 1. Purpose

Ledger is an embedded key-value storage engine written in Ada and SPARK. Its defining property is that a write the engine has acknowledged survives a crash, and that the core of this claim is backed by machine-checked proof, not by tests alone.

Each of the three Ledger teams builds the full engine. This document describes what Team A commits to build and how each commitment will be checked.

## 2. Scope

After the Sprint 1 meetings with the Product Owner the team narrowed the scope to a small prototype that puts data retention and retrieval after a failure first. Everything below follows from that.

**In scope**

- A library that stores key-value pairs in files on a local disk.
- Put, Get and Delete of single keys.
- Recovery after a crash at any instant.
- A checkpoint that keeps the log and the recovery time bounded.
- Concurrent readers next to a single writer (late sprint, first to be cut if the schedule slips).
- A small command-line tool used to demo the engine at sprint reviews.

**Out of scope this semester**

- Transactions that span several keys.
- More than one writer at a time.
- A query language, networking, replication, encryption, compression.
- Values larger than the fixed limit in section 5.

## 3. Stakeholders

| Who | What they need from Ledger |
|---|---|
| Product Owner | An end-to-end path that works and a proof story he can inspect |
| Course instructor | Evidence on the board for each sprint: requirements, design, working increments |
| A developer embedding Ledger | A small API whose failure behaviour is fully specified |

## 4. Definitions

| Term | Meaning in this project |
|---|---|
| Crash | The process stops at an arbitrary instant: kill signal or power loss. Writes that were not synced may be lost or partially written. |
| Acknowledged write | A Put or Delete that returned `Success` to the caller. |
| Durable | Still present after any later crash and recovery. |
| Torn write | A write that reached the disk only in part. |
| WAL | Write-ahead log: the file every change is appended to before it is considered committed. |
| Checkpoint | Copying committed changes from the WAL into the data file, then emptying the WAL. |
| Recovery | The procedure run on open that brings the files back to a consistent state. |

## 5. Functional requirements

| ID | Requirement | Verified by |
|---|---|---|
| FR-1 | `Open` creates a new database or opens an existing one at a given path. Recovery always runs before any other operation is served. | Test |
| FR-2 | `Put (Key, Value)` stores the value, replacing any previous value for that key. | Test |
| FR-3 | `Get (Key)` returns the most recent acknowledged value, or `Not_Found`. | Test |
| FR-4 | `Delete (Key)` removes the key. Deleting a missing key is not an error. | Test |
| FR-5 | **Durability.** Once Put or Delete returns `Success`, its effect survives any later crash. | Crash test |
| FR-6 | **Atomicity.** After recovery, an operation that was interrupted by a crash is either fully applied or not applied at all. It is never partly visible. | Crash test |
| FR-7 | **Recovery is idempotent.** If a crash happens during recovery, running recovery again reaches the same state as an uninterrupted recovery. | Proof (Gold) and crash test |
| FR-8 | **Corruption is detected.** Every WAL record carries a checksum. An incomplete or corrupted record at the tail of the WAL is detected and discarded, never applied. | Proof of the codec (Gold) and test |
| FR-9 | **Checkpoint.** Committed changes are moved into the data file and the WAL is emptied, so the WAL does not grow without bound. | Test |
| FR-10 | **Readers.** Several readers may call Get while one writer is active. Readers only ever see committed state. | Test (Sprint 5) |
| FR-11 | Every public operation reports failure through a status value. No exception escapes the public API. | Proof (Silver) |
| FR-12 | A command-line tool exposes put, get, delete and a crash-and-recover demonstration. | Demo |

**Fixed limits (initial values, defined once as constants)**

| Limit | Value |
|---|---|
| Key length | 1 to 32 bytes |
| Value length | 0 to 128 bytes |
| Page size | 4096 bytes |
| Writers | 1 |
| Processes with the database open | 1 |

Fixed sizes are a deliberate choice. They let the core use bounded arrays with no heap allocation, which keeps the proofs within reach of the team (see `decisions/0003-proof-scope.md`).

## 6. Non-functional requirements

| ID | Requirement | Verified by |
|---|---|---|
| NFR-1 | Built with Alire. `alr build` succeeds from a clean clone with no manual steps. | CI |
| NFR-2 | **SPARK Silver everywhere.** For every unit in SPARK mode, GNATprove reports no unproved checks: no overflow, no out-of-range index, no failed precondition. | GNATprove report |
| NFR-3 | **SPARK Gold for the core.** Two properties are proved: WAL record decode is the inverse of encode, and applying the committed log twice gives the same result as applying it once. | GNATprove report |
| NFR-4 | Code outside SPARK is confined to one I/O boundary package. That package is kept small and its assumptions are listed in the design document. | Review |
| NFR-5 | **Crash testing.** A fault-injection harness simulates a crash at every write and sync point of a workload. After each simulated crash, recovery must produce a state equal to a reference model of the acknowledged operations. | Crash test suite |
| NFR-6 | **Recovery time** is measured as a function of WAL size and reported. The numeric target is set after the first measurement in Sprint 4. | Benchmark |
| NFR-7 | No heap allocation in SPARK units. All structures are bounded. | Review, GNATprove |
| NFR-8 | Every change reaches `main` through a pull request reviewed by one other team member. From Sprint 3, CI runs build, tests and proof on each pull request. | Repository history |

## 7. Assumptions

The proofs and the tests rest on these. If one of them is false, the guarantees do not hold.

1. When the operating system reports a sync as complete, the data is on stable storage.
2. Storage does not silently change data that was synced. Damage to the unsynced tail of the WAL is expected and handled (FR-8).
3. One process has exclusive access to the database files.
4. The files live on one local file system.

## 8. Open questions for the Product Owner

1. Are the key and value limits in section 5 acceptable for the prototype?
2. Is an ordered range scan required, or is point lookup enough?
3. Is rebalancing on delete required, or may the prototype leave under-full pages (see design section 6.3)?
4. Is there a recovery-time target you expect, or do we propose one from measurements?
5. What is the minimum proof level you expect to see at the final review?

## 9. Traceability

| Requirement | Design section | Board card |
|---|---|---|
| FR-1, FR-7, FR-8 | Design 5, 7 | Plan / Build the write-ahead log |
| FR-2, FR-3, FR-4 | Design 6 | Plan / Build the B-tree |
| FR-5, FR-6, NFR-5 | Design 5, 9 | Create crash tests |
| FR-9 | Design 8 | Add checkpointing |
| FR-10 | Design 10 | Add support for multiple readers |
| NFR-6 | Design 8, 9 | Test recovery speed |
| NFR-1, NFR-2, NFR-3 | Design 9, decision 0003 | Set up Ada and SPARK |
