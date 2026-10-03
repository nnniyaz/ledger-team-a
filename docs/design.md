# Ledger: Design

| | |
|---|---|
| Team | Team A, FIU Capstone 2, Fall 2026 |
| Version | 0.1 draft, 2026-10-02 |
| Status | Written by the Team Leader. Pending review by the team and the Product Owner. |
| Reads with | `requirements.md`, `decisions/0001` to `0003`, `sdlc.md` |

## 1. Summary

Ledger is a paged key-value store. Keys and values live in a B+tree whose nodes are fixed-size pages in a data file. Every change is first appended to a write-ahead log (WAL) and synced; only then is the operation acknowledged. The data file is written only during a checkpoint. After a crash, recovery replays the committed part of the WAL into the data file.

Three decisions shape this design. Each has its own record with the alternatives that were rejected and the criterion used:

| Decision | Chosen | Record |
|---|---|---|
| How data survives a crash | Write-ahead log | `decisions/0001-durability.md` |
| Index structure | B+tree | `decisions/0002-index.md` |
| What is proved | Silver everywhere, Gold for the WAL codec and recovery | `decisions/0003-proof-scope.md` |

The criterion in all three is the same: **what four engineers who are new to Ada and SPARK can both implement and prove in the three sprints that remain.**

## 2. Architecture

```
            caller
              |
      +----------------+
      | Ledger.Engine  |   public API: Open, Put, Get, Delete, Checkpoint, Close
      +----------------+
              |
      +----------------+
      | Ledger.Btree   |   keys and values in pages
      +----------------+
              |
      +----------------+        +------------------+
      | Ledger.Pager   |------->| Ledger.Wal       |   append, sync, scan
      | page cache     |        | Ledger.Wal_Codec |   record encode / decode
      +----------------+        +------------------+
              |                          |
      +------------------------------------------+
      | Ledger.IO   (the only unit outside SPARK) |   read, write, sync, truncate
      +------------------------------------------+
              |
        data file, WAL file

      Ledger.Recovery uses Wal and IO on Open, before the Engine serves anything.
```

Each layer talks only to the layer directly below it. This matters for two reasons: the durability argument stays inside `Wal`, `Wal_Codec` and `Recovery`, independent of the B+tree, and four people can work on four units in parallel.

## 3. Units

| Unit | Responsibility | SPARK |
|---|---|---|
| `Ledger.Types` | Page, key, value, page number, LSN and status types; the size constants | Yes |
| `Ledger.Wal_Codec` | Pure functions: record to bytes, bytes to record, checksum | Yes, Gold |
| `Ledger.Wal` | Append records, sync, scan the log from the start | Yes, Silver |
| `Ledger.Recovery` | Select committed records and apply them to the data file | Yes, Gold for idempotence |
| `Ledger.Pager` | Page cache, dirty tracking, page allocation, checkpoint | Yes, Silver |
| `Ledger.Btree` | Search, insert with split, delete | Yes, Silver |
| `Ledger.Engine` | Public API, single-writer discipline | Yes, Silver |
| `Ledger.IO` | Thin binding to the operating system; spec has SPARK contracts, body is not in SPARK | Spec only |
| `Ledger.IO.Faulty` | Test-only implementation of the same spec that injects crashes | No |

## 4. Files on disk

Two files per database: `<name>.db` (data) and `<name>.wal` (log).

### 4.1 Data file

An array of 4096-byte pages. Page 0 is the meta page.

| Meta page field | Size (bytes) | Meaning |
|---|---|---|
| Magic | 8 | Identifies a Ledger data file |
| Format version | 4 | |
| Page size | 4 | 4096 |
| Root page | 4 | Page number of the B+tree root |
| Page count | 4 | Number of pages allocated so far |

The meta page changes through the WAL like every other page.

### 4.2 WAL file

A sequence of records, appended only.

| Record field | Size (bytes) | Meaning |
|---|---|---|
| LSN | 8 | Log sequence number, increases by one per record |
| Kind | 1 | `Page_Image` or `Commit` |
| Page number | 4 | Target page; 0 for `Commit` |
| Length | 2 | Payload length: 4096 for `Page_Image`, 0 for `Commit` |
| Payload | Length | Full image of the page after the change |
| Checksum | 4 | CRC-32 over all fields above |

The log is **redo-only** and stores **full page images**. Both choices are for provability:

- Redo-only: the data file is never written before commit, so there is nothing to undo.
- Full page images: applying a record means "set page N to this image". Doing that twice gives the same file as doing it once. This is what makes recovery idempotent, and it also makes a torn page write in the data file harmless, because the full image is still in the WAL.

The cost is log volume: one small Put writes at least one full page to the log. For a prototype whose goal is provable crash safety this is an accepted trade.

## 5. Write path

`Put (Key, Value)` and `Delete (Key)` are each one transaction.

1. `Btree` changes pages through `Pager`. The pager changes its cached copies and marks them dirty. The data file is not touched.
2. `Pager` appends one `Page_Image` record per dirty page to the WAL.
3. `Pager` appends a `Commit` record.
4. `Wal` syncs the WAL file.
5. The operation returns `Success`.

The operation is acknowledged only after step 4. A crash before the sync completes leaves no `Commit` record that recovery will accept, so the operation never happened. A crash after it leaves a complete committed group, so recovery will apply it. This is requirements FR-5 and FR-6.

**Pager invariant.** A page whose latest committed image is in the WAL and not yet in the data file is always present in the cache. Dirty pages are not evicted before a checkpoint. Because of this, reads never have to search the WAL.

## 6. B+tree

Values live only in leaves. Internal nodes hold separator keys and child page numbers.

### 6.1 Page layout

Slots have a fixed size, derived from the limits in the requirements (key up to 32 bytes, value up to 128 bytes).

| Node kind | Slot content | Approximate capacity per page |
|---|---|---|
| Leaf | key length, key, value length, value | 25 entries |
| Internal | key length, key, child page number | 110 separators |

Fixed slots waste space with short keys. In exchange, a node is a plain bounded array, and the "keys are sorted and count is within bounds" invariant can be stated directly as a SPARK type predicate.

### 6.2 Operations

- **Search:** walk from the root, binary search inside each node.
- **Insert:** insert into the leaf; if the leaf is full, split it and insert the separator into the parent, repeating upward. A root split allocates a new root and updates the meta page.
- **Delete:** remove the entry from its leaf.

### 6.3 Known limitation

Delete does not merge or rebalance under-full pages, and pages are never returned to a free list. The tree stays correct; it may use more space than necessary after many deletes. Rebalancing is the most proof-heavy part of a B+tree and is left out on purpose. This is listed as an open question for the Product Owner in the requirements.

## 7. Recovery

Runs inside `Open`, before anything else.

1. Scan the WAL from the start. For each record check the checksum and that the LSN continues the sequence. Stop at the first record that is incomplete or fails either check. Everything after that point is discarded.
2. Keep only the records that belong to a group closed by a `Commit` record. A trailing group without `Commit` is discarded.
3. Apply the kept `Page_Image` records to the data file in log order: write the image at offset `page number * 4096`.
4. Sync the data file.
5. Truncate the WAL to empty and sync it.

**Why a crash during recovery is safe**

| Crash during | State on disk | Next recovery |
|---|---|---|
| Steps 1 and 2 | Nothing was written | Starts over |
| Step 3 or 4 | Data file partly updated, WAL intact | Applies the same images again; same result |
| Step 5 | WAL is either intact or empty | Applies again, or has nothing to do |

The WAL is emptied only after the data file is synced, so at every instant at least one complete copy of each committed page exists.

## 8. Checkpoint

A checkpoint is the same three final steps as recovery, with the cache as the source: write every committed dirty page to the data file, sync the data file, truncate and sync the WAL. The crash argument is the table above.

A checkpoint runs when the WAL passes a size threshold or the cache has no clean page left to reuse. It runs in the writer, between operations. Recovery time is bounded by the WAL size threshold, which is the lever for NFR-6.

## 9. Verification

Three mechanisms, each for the kind of claim it is suited to.

### 9.1 Proof

| Level | Where | What it shows |
|---|---|---|
| Silver | Every SPARK unit | No run-time errors: no overflow, no index out of range, no violated precondition |
| Gold | `Wal_Codec` | `Decode (Encode (R)) = R` for every valid record; a buffer with a wrong checksum is rejected |
| Gold | `Recovery` | Over a model of the data file as an array of pages, applying the committed log twice equals applying it once |

### 9.2 Crash tests

Proof stops at the `Ledger.IO` spec. What happens below it (the operating system, the disk) is tested, not proved.

`Ledger.IO.Faulty` implements the same spec in memory. It counts I/O calls and at call number N simulates a crash: writes that were not synced are dropped, and the last write may be left as a torn prefix. The harness:

1. Runs a workload once to count its I/O calls.
2. For each N from 1 to that count: runs the workload, crashes at call N, runs recovery, then compares the engine's contents with a reference model.

The reference model is an in-memory map of the acknowledged operations. The check is: every acknowledged operation is present, and the one operation in flight at the crash is either fully present or fully absent.

A second pass crashes during recovery itself, to exercise the table in section 7.

### 9.3 Unit tests and benchmark

Ordinary tests for the B+tree (insert, split, lookup, delete) and for the engine API. A benchmark measures recovery time against WAL size.

## 10. Readers (Sprint 5)

One writer, many readers. The plan is a protected object around the pager so that a reader sees a page either before or after a committed change, never in between. This is the last feature on the schedule and the first to be cut. The detailed design is deferred until the single-threaded engine passes its crash tests.

## 11. Assumptions at the I/O boundary

`Ledger.IO` is the trusted base. The design relies on:

1. A completed sync means the data is on stable storage.
2. A write that was not synced may be lost entirely or in part, but does not damage data that was synced earlier.
3. Truncating a file to empty either happens or does not.

The crash tests model exactly these three behaviours.

## 12. Build order

| Sprint | Increment | End-to-end path that can be shown |
|---|---|---|
| 2 (now) | Requirements, design, decisions; Alire project that builds and passes GNATprove | Toolchain |
| 3 | `Types`, `Wal_Codec` with Gold proof, `Wal`, `IO`, `IO.Faulty`, first crash tests | Append records, crash, recover the committed ones |
| 4 | `Pager`, `Btree`, `Recovery` with Gold proof, checkpoint, CLI | Put, crash, reopen, Get |
| 5 | Readers, recovery benchmark, remaining proof work, showcase material | Full demo |
