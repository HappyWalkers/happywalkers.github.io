---
title: "Inside BusTub: How CMU's Teaching Database Fits Together"
description: "The architecture of BusTub, the C++ database system behind CMU 15-445 (Fall 2025), layer by layer: buffer pool, ARC eviction, table heap, B+tree, planning, execution and MVCC, with a diagram for each."
tags:
  - Databases
  - C++
thumbnail: /assets/images/thumbnails/bustub.jpg
---

I have finished reading through [BusTub](https://github.com/cmu-db/bustub), the relational database that CMU's [15-445/645 Intro to Database Systems](https://15445.courses.cs.cmu.edu/fall2025/) is built around, with all five Fall 2025 projects filled in. BusTub is small enough to read end to end, yet it has the parts of a real system: a SQL front end, a rule-based optimizer, a batched executor engine, a concurrent B+tree, a buffer pool and multi-version concurrency control.

The course asks students [not to publish their implementations](https://15445.courses.cs.cmu.edu/fall2025/faq.html), so this post contains no project code. It covers the architecture and the ideas behind each layer, at the level of detail the public starter code and project write-ups already give, and uses BusTub's own class names so it can be read next to the public repository.

<!--more-->

<style>
.bt-fig { margin: 1.75rem 0; }
.bt-fig a { display: block; }
.bt-fig img { display: block; width: 100%; height: auto; margin: 0 !important; border: 1px solid var(--line); }
.bt-fig figcaption { margin-top: 0.6rem; color: #666; font-size: 0.85rem; line-height: 1.5; }
@media (max-width: 640px) {
  .bt-fig a { overflow-x: auto; -webkit-overflow-scrolling: touch; }
  .bt-fig img { width: 620px; max-width: none; }
}
</style>

## The whole system {#overview}

The Project 3 write-up has [a one-page diagram of BusTub](https://15445.courses.cs.cmu.edu/fall2025/project3/img/project-structure.svg): parser, binder, planner and optimizer on top, executors in the middle, then the transaction manager, the index and the table heap, and finally the buffer pool and the disk manager. Figure 1 keeps that layout and fills in what sits inside each box. The rest of the post zooms into one layer at a time, from the disk up, which is also the order in which the projects build the system.

<figure class="bt-fig">
<a href="/assets/images/posts/bustub/overview.svg"><img src="/assets/images/posts/bustub/overview.svg" alt="BusTub architecture in four bands. Query processing: SQL goes through the parser (libpg_query), binder, planner and optimizer, with the catalog on the side. Query execution: a tree of executors such as SeqScan, IndexScan, Insert, Update, Delete, Aggregation, joins, ExternalMergeSort, Limit and WindowFunction. Concurrency control: the transaction manager, and a disabled lock manager. Storage: the B+tree index, the table heap and temporary pages, all on top of the buffer pool manager (page table, frames, ARC replacer, page guards), the disk scheduler, the disk manager and the database file."></a>
<figcaption markdown="span">Figure 1. BusTub's layers and the project that builds each part, extended from the course's diagram. P0, a count-min sketch, is a C++ warm-up that is not wired into the system. On a phone, swipe a figure sideways or tap it to open it full size.</figcaption>
</figure>

A query enters through `BusTubInstance::ExecuteSql`, and `ExecuteSqlTxn` in `src/common/bustub_instance.cpp` is the best single function to start reading from. It binds the statement and handles `CREATE`, `EXPLAIN`, `SET` and `BEGIN`/`COMMIT` directly. Everything else is planned, optimized, turned into an executor tree and run, and the rows that come out are printed through a `ResultWriter`.

Two subsystems in the source tree are switched off this semester. `LockManager` (two-phase locking with deadlock detection) is compiled but unused because Project 4 does concurrency control with MVCC instead, and `LogManager`/`CheckpointManager` exist without being enabled. The `Catalog` is an in-memory map, too, so tables do not survive a restart: BusTub has no crash recovery. Logging and recovery are covered in lectures and homework only.

## Pages and the buffer pool {#buffer-pool}

A database file is a sequence of fixed-size pages, 8 KiB in BusTub (`BUSTUB_PAGE_SIZE`). The lectures argue that the DBMS should cache those pages itself rather than rely on the operating system's page cache through `mmap`: it knows its own access patterns, and it has to decide when a dirty page may reach the disk. The buffer pool is that cache, and in BusTub it is the only path to the disk. B+tree nodes, table pages and the temporary pages of sorts and hash joins are all requested by page id.

<figure class="bt-fig">
<a href="/assets/images/posts/bustub/buffer-pool.svg"><img src="/assets/images/posts/bustub/buffer-pool.svg" alt="The buffer pool manager. Callers (B+tree, table heap, executors) pass a page id and receive a page guard. Inside: the page table maps page ids to frames, an array of frames each with a page, pin count and dirty flag, an ArcReplacer that picks a pin-0 frame to evict, and a free-frame list. Below, the DiskScheduler queues read and write requests on a channel; a background worker thread runs them against the DiskManager and fulfills each request's promise."></a>
<figcaption markdown="span">Figure 2. `BufferPoolManager` and `DiskScheduler`. Frame 1 has a pin count of 0, so the replacer may evict it; it is dirty, so it must be written back first.</figcaption>
</figure>

A frame is a `FrameHeader`: 8 KiB of data, an atomic pin count, a dirty flag and a reader-writer latch. The page table maps the page ids that are currently resident to frames. A request for a page that is not resident goes through the textbook steps:

1. Take a frame from the free list or, if there is none, ask the replacer for a victim among the unpinned frames.
2. If the victim is dirty, write it back, then drop its page-table entry.
3. Read the requested page into the frame, map it, pin it and record the access.

When every frame is pinned there is nothing to evict, which is why `CheckedReadPage` and `CheckedWritePage` return a `std::optional`.

Three words that sound alike mean different things here. A pin says "do not evict this frame". A latch is a short-term mutex that protects the frame's bytes from other threads. A lock belongs to a transaction and protects logical data for the length of that transaction; the buffer pool never deals with locks. The API combines the first two into `ReadPageGuard` and `WritePageGuard`, which hold a pin plus a shared or exclusive latch and release both when they are destroyed. They are move-only RAII types, so ownership of a page is visible in the code. Because a guard keeps its frame pinned, holding one longer than needed shrinks the pool for every other thread.

The `DiskScheduler` keeps blocking I/O off the callers' threads. `Schedule` takes a vector of `DiskRequest`s (read or write, a pointer into the frame, a page id, and a `std::promise<bool>`) and puts them on a channel. One background worker takes requests off the channel, runs them through the `DiskManager` and sets each promise, while the caller waits on the matching future. Accepting a vector of requests leaves room for prefetching and parallel I/O, which is what the Project 1 leaderboard rewards.

## ARC: choosing what to evict {#arc}

Plain LRU fails on sequential flooding: one large scan pushes every hot page out of the pool, even though the scan will never touch those pages again. LRU-K, 2Q and ARC address this by telling recency apart from frequency. Fall 2025 uses a variant of [ARC](https://www.cs.cmu.edu/~natassa/courses/15-721/papers/arcfast.pdf) (adaptive replacement cache) from IBM, specified step by step in the [Project 1 write-up](https://15445.courses.cs.cmu.edu/fall2025/project1/).

<figure class="bt-fig">
<a href="/assets/images/posts/bustub/arc-replacer.svg"><img src="/assets/images/posts/bustub/arc-replacer.svg" alt="ARC's four lists in a row: MRU ghost (page ids only), MRU (frames seen once), MFU (frames seen twice or more) and MFU ghost (page ids only). With an MRU target size of 2 and three entries in MRU, Evict takes the MRU tail and moves its page id to the MRU ghost list. Below are the four RecordAccess cases and the Evict rules."></a>
<figcaption markdown="span">Figure 3. The two resident lists sit between their ghost lists. The resident lists hold at most one entry per frame; all four lists together hold at most twice that.</figcaption>
</figure>

The MRU list holds pages that were accessed once recently, the MFU list pages that were accessed more than once, and each has a ghost list of page ids that were recently evicted from it. A ghost hit is a near miss: had that list been a little larger, the page would still be cached. ARC therefore shifts the MRU list's target size toward the list whose ghost was hit (an MRU-ghost hit raises it, an MFU-ghost hit lowers it), and `Evict` takes from MRU while MRU is at or above its target and from MFU otherwise. The policy tunes itself between scan-heavy and reuse-heavy workloads without a parameter.

The replacer tracks frames and pages together because the ghost lists outlive the frame. A ghost entry has to be recognized by its page id after the frame holds another page. `RecordAccess` also receives an `AccessType` (lookup, scan or index). The required algorithm ignores it, but the leaderboard section suggests using it, for example to keep scans from flooding the pool.

## Tables: slotted pages in a heap {#table-heap}

A table is a heap: an unordered collection of pages. Each page is slotted, with a slot array growing from the front and tuple bytes growing from the back, so a record id of the form (page, slot) stays valid even if the bytes move within the page.

<figure class="bt-fig">
<a href="/assets/images/posts/bustub/table-heap.svg"><img src="/assets/images/posts/bustub/table-heap.svg" alt="TableHeap as a linked list of pages 3, 8 and 11 from first_page_id_ to last_page_id_. Page 8 is expanded: a header with next_page_id_, num_tuples_ and num_deleted_tuples_, then slots 0 to 2 each holding an offset, a size and a TupleMeta, free space, and tuples 2, 1 and 0 packed from the end of the page. A RID (page 8, slot 1) points to slot 1. Notes: TupleMeta holds a timestamp and an is_deleted flag; a Tuple holds serialized values and its RID but no schema."></a>
<figcaption markdown="span">Figure 4. `TableHeap` and `TablePage`. Header sizes are in bytes; each slot also carries the tuple's `TupleMeta`.</figcaption>
</figure>

`TableHeap` links its `TablePage`s through `next_page_id_` and appends new tuples to the last page, adding a page when that one is full. Each slot carries a `TupleMeta`: a timestamp, which Project 4 turns into the version stamp, and an `is_deleted_` flag. Deleting a tuple only sets the flag, and space is never reclaimed, so a RID stays valid for the life of the table. Indexes rely on that, because they store bare RIDs.

A `Tuple` is serialized bytes plus its RID. It does not know its schema: every `GetValue` call is told which schema to read with. Plan nodes therefore carry their output schemas, and an executor that reads with the wrong schema gets wrong values rather than a helpful error.

The iterator from `TableHeap::MakeIterator` remembers the last tuple that existed when it was created and stops there. That avoids the Halloween problem: in Project 3 an update is a delete followed by an insert, and without the stopping point a scan feeding an update would keep finding the rows the update had just inserted.

## The B+tree index {#bplustree}

A B+tree keeps keys sorted in a balanced tree whose inner nodes only route searches and whose leaves hold the entries and are chained left to right for range scans. Nodes split when they overflow and merge or borrow from a sibling when they fall below half full, so every lookup costs the same number of page reads.

<figure class="bt-fig">
<a href="/assets/images/posts/bustub/bplustree.svg"><img src="/assets/images/posts/bustub/bplustree.svg" alt="A header page holding root_page_id_ points to a root internal page with keys (unused), 20 and 40 and three children. Three leaves hold keys 5, 9, 13; 20, 27, 33; and 40, 48, linked by next pointers, the last one INVALID. Key 27 is struck through as a tombstone. An IndexIterator arrow runs along the leaves. The middle leaf is expanded: header fields, a tombstone buffer holding index 1, the key array 20, 27, 33 and the RID array (4,0), (4,1), (7,2)."></a>
<figcaption markdown="span">Figure 5. B+tree pages. Every node is a buffer-pool page, read through a page guard and cast to `BPlusTreeInternalPage` or `BPlusTreeLeafPage`.</figcaption>
</figure>

Three things in BusTub's tree are easy to miss when coming from a textbook:

- **A separate header page holds the root's page id.** When the root splits or collapses, changing the root is an ordinary page write under the header page's latch, so concurrent threads see a consistent root.
- **An internal page stores keys and child page ids as pairs.** A node with *n* + 1 children has only *n* separator keys, so the key in slot 0 is unused. Leaves pair keys with RIDs, and only unique keys are allowed.
- **Leaves have a tombstone buffer** (new in Fall 2025, a simplified version of the Bε-tree from lecture). A delete records the entry's index in a small fixed buffer instead of removing it; only when the buffer is full is the oldest delete applied. Lookups, the iterator and every split or merge must respect the tombstones.

The tree plugs into the rest of the system through the `Index` interface, three virtual functions over (key tuple, RID) pairs: `InsertEntry`, `DeleteEntry` and `ScanKey`. `BPlusTreeIndex` adapts it to the tree, and `IndexIterator` walks the leaf chain to serve ordered scans.

## Latch crabbing {#latch-crabbing}

Concurrent operations use latch coupling ("crabbing"): take the child's latch before releasing the parent's, so no other thread can change the path in between. A node is *safe* when the current operation cannot spread past it, meaning an insert will not split it or a delete will not make it underflow. Once a safe node is latched, every ancestor latch can be released.

<figure class="bt-fig">
<a href="/assets/images/posts/bustub/latch-crabbing.svg"><img src="/assets/images/posts/bustub/latch-crabbing.svg" alt="Three panels, each a path of header, root, inner and leaf pages. Search: read latches, only the leaf still held. Optimistic insert: read latches on the way down and a write latch on the leaf only. Insert when the leaf is full: restart with write latches; the inner page has room, so header and root are released while inner and leaf stay write-latched."></a>
<figcaption markdown="span">Figure 6. The optimistic version of latch crabbing that Project 2 asks for. Latches are only taken from parent to child, so two threads descending the tree cannot wait on each other in a cycle.</figcaption>
</figure>

Taking write latches from the root down for every insert would serialize all writers on the root. The optimistic variant bets that the leaf is safe, which it usually is: it descends with read latches, takes a write latch only on the leaf, and restarts with write latches from the top only when the bet fails. The header file provides a `Context` struct (the header-page guard, the root page id, and deques of the read and write guards along the path) for this bookkeeping.

## From SQL to a plan {#planning}

The front end is provided apart from a few optimizer rules, and reading it is the quickest way to understand what an executor receives. `EXPLAIN` prints every stage, so Figure 7 follows one join through them. The plans are real `bustub-shell` output, trimmed, except for the last one, which follows the hash-join form shown in the Project 3 write-up.

<figure class="bt-fig">
<a href="/assets/images/posts/bustub/planning.svg"><img src="/assets/images/posts/bustub/planning.svg" alt="SELECT * FROM t1, t2 WHERE t1.a = t2.a AND t1.b = t2.c goes through four stages. Parser and binder produce a BoundSelect over a cross product with the where clause. The planner produces Projection over Filter over a NestedLoopJoin with predicate true over two SeqScans. The starter optimizer rules remove the projection and merge the filter into the join predicate. The NLJAsHashJoin rule turns it into a HashJoin with left_key [#0.0, #0.1] and right_key [#0.0, #0.1]. ExecutorFactory then builds one executor per plan node."></a>
<figcaption markdown="span">Figure 7. Binding, planning and rule-based optimization of a two-column equi-join.</figcaption>
</figure>

The binder hands the text to [libpg_query](https://github.com/pganalyze/libpg_query), PostgreSQL's own parser, and converts its parse tree into BusTub's `BoundStatement` classes, resolving every table and column against the catalog. The planner turns that into a tree of plan nodes, each with an output schema, and expressions into `AbstractExpression` trees: `ColumnValueExpression` (`#i.j`), comparisons, arithmetic, logic and constants.

The optimizer is rule-based, with no cost model and no search over join orders. A rule is a function that optimizes the children first and then rebuilds the node with `CloneWithChildren`, because plan nodes are immutable and shared. Students add rules such as `NLJAsHashJoin` and `SeqScanAsIndexScan` in Project 3, and leaderboard rules such as predicate pushdown and column pruning. Without statistics, `EstimatedCardinality` guesses a table's size from a suffix of its name (`_1k`, `_100k`, `_1m`), the only size information a rule can use. Some tests run `SET force_optimizer_starter_rule=yes` so that only the starter rules apply.

## Executing a plan {#execution}

The lectures compare three processing models: the iterator (Volcano) model pulls one tuple at a time, the materialization model passes whole results between operators, and the vectorized model passes batches. Fall 2025 BusTub is vectorized. `ExecutorFactory` mirrors the plan tree into an executor tree, and every executor implements `Init()` and `Next(&tuples, &rids, batch_size)`. `Next` fills up to `batch_size` tuples with their RIDs (the engine asks for `BUSTUB_BATCH_SIZE`, 20) and returns `false` once the executor is exhausted.

<figure class="bt-fig">
<a href="/assets/images/posts/bustub/execution.svg"><img src="/assets/images/posts/bustub/execution.svg" alt="Left: the executor tree for SELECT a, COUNT(*) FROM t1 GROUP BY a ORDER BY a LIMIT 3, which is Limit over ExternalMergeSort over Aggregation over SeqScan over the TableHeap iterator. Next calls go down and batches of up to 20 tuples and RIDs come up; ExternalMergeSort and Aggregation are marked as pipeline breakers. Right top: a two-way external merge sort, where pass 0 sorts each page, pass 1 merges pairs of runs and pass 2 produces one sorted run. Right bottom: a Grace hash join that partitions t1 and t2 with hash h1 and joins each pair of partitions with an h2 hash table."></a>
<figcaption markdown="span">Figure 8. Left: a real optimized plan run as a pull-based tree. Right: the two Project 3 operators that must handle inputs larger than memory.</figcaption>
</figure>

`ExecutionEngine` calls `Init()` on the root and then `Next()` until it returns `false`. Most executors stream: a filter or a projection hands on each batch as soon as it has one. Pipeline breakers such as aggregation, sorting and the build side of a hash join must consume their whole input before they can emit a single row. The executors fall into groups:

- **Access methods:** `SeqScan` and `IndexScan` (a point lookup with `ScanKey`, or an ordered scan with the index iterator).
- **Modifications:** `Insert`, `Update` and `Delete` output a single row with a count, and they also update every index on the table.
- **Joins:** `NestedLoopJoin` rescans the inner side for every outer tuple, `NestedIndexJoin` probes an index on the inner side instead, and `HashJoin` builds a hash table once.
- **Others:** hash-based `Aggregation` (with SQL's `NULL` rules), `ExternalMergeSort`, `Limit` and `WindowFunction`.

Two Project 3 operators may not assume that their input fits in memory. A hash join follows the Grace algorithm: partition both inputs with one hash function, then join matching partitions with an in-memory table built with a second one. `ORDER BY` without a usable index runs a two-way external merge sort, which sorts each page and then merges pairs of runs until one is left. Both store intermediate data in `IntermediateResultPage`s, a page layout the student designs, allocated through the buffer pool and deleted when no longer needed. Spilling therefore needs no special I/O code: the buffer pool decides when a page actually goes to disk.

The tests are mostly [SQLLogicTest](https://www.sqlite.org/sqllogictest/) files (`test/sql/*.slt`), which run SQL and compare the output with expected rows. A few also check behavior, such as whether a nested-loop join really re-initializes its inner side.

## Transactions with MVCC {#mvcc}

Project 4 adds transactions with [optimistic multi-version concurrency control](https://15445.courses.cs.cmu.edu/fall2025/project4/), a protocol that, as the write-up notes, HyPer and DuckDB use in a similar form. Readers never block writers: each transaction reads a snapshot, and older versions are rebuilt on demand from undo records. The table heap does not change shape; the versioning lives beside it.

<figure class="bt-fig">
<a href="/assets/images/posts/bustub/mvcc.svg"><img src="/assets/images/posts/bustub/mvcc.svg" alt="Three columns. The table heap holds the newest version of RIDs 0/0 (ts 4, (1, 90)), 0/1 (ts txn9, (2, 70)) and 0/2 (ts 3, (3, 20), deleted). version_info_ in the TransactionManager maps each slot to the head undo log: txn7@0, txn9@0 and txn6@0. The transactions hold undo logs: txn7, committed at 4, holds (_, 100) with ts 2, pointing to txn4@0; txn9, running with read_ts 4, holds (_, 50) with ts 3; txn6, committed at 3, holds (3, 20) with ts 1; txn4, committed at 2, holds (_, 120) with ts 1 and is drawn faded; txn8 is running with read_ts 2 and has no writes. A table shows what each reader sees, and a note explains that the watermark is 2, so txn4 can be garbage-collected."></a>
<figcaption markdown="span">Figure 9. An example of my own in the storage format of Project 4: the newest versions in the heap, one link per slot in the transaction manager, and older versions as deltas in the workspaces of the transactions that overwrote them.</figcaption>
</figure>

**Timestamps.** A transaction gets a read timestamp when it begins (the latest commit timestamp) and a commit timestamp when it commits; commit timestamps define the serialization order. A version written by a transaction that has not committed yet carries a temporary timestamp: the writer's transaction id. Transaction ids count up from `TXN_START_ID` (2<sup>62</sup>), so txn9 is 2<sup>62</sup> + 9, larger than any commit timestamp, and "is this version committed?" is a single comparison.

**Where versions live.** There are three places: the heap holds the newest version and its timestamp; `TransactionManager::version_info_` holds, for every page and slot, an `UndoLink` to the newest older version; and each transaction keeps the `UndoLog`s it created. An undo log is a delta: a bitmap of the modified columns, a partial tuple with their old values, the timestamp of the version it restores, and a link to the next older log. Following the links from a slot gives the version chain.

**Reading.** If the heap version is not newer than the reader's read timestamp, or the reader wrote it itself, the reader uses it. Otherwise it walks the chain, applying deltas, until it reaches a log whose timestamp is at or below its read timestamp. If the chain ends first, the tuple did not exist yet for that reader. The table in Figure 9 shows the three results the same heap gives three readers.

**Writing.** A write conflicts with another when the tuple's newest version belongs to another uncommitted transaction or was committed after the writer's read timestamp. The writer is then marked `TAINTED` and aborts: the first writer wins, and nothing waits. Otherwise the writer records an undo log (one per tuple per transaction, merged on later writes) and installs the new tuple and the new head link together. `UpdateTupleAndUndoLink` does that under the page latch and runs a check callback first, so a writer can re-validate the tuple at the moment it installs the new version; without it, two writers could both pass the conflict check. At commit, under a commit mutex, the transaction stamps every tuple in its write set with its commit timestamp. Aborting restores the previous version.

**Garbage collection.** The watermark is the lowest read timestamp among running transactions, and no reader will ever need a version older than what the watermark sees. A finished transaction whose undo logs are all hidden behind newer ones can be dropped from the transaction map. `Watermark` keeps the running read timestamps in a structure that answers the minimum quickly instead of scanning every transaction.

**Indexes and serializability.** With MVCC an index entry has to outlive the tuple's deletion, because older snapshots can still reach the tuple through the index. A primary-key entry therefore always points to the same RID: re-inserting a deleted key reuses the old slot, and changing a primary key is a delete plus an insert. Snapshot isolation is the default. For `SERIALIZABLE`, a committing transaction runs optimistic backward validation: it checks the scan predicates it recorded against the writes of every transaction that committed after its read timestamp, and aborts if any overlap.

## Tools that help with reading the code {#tools}

- `bustub-shell` runs SQL interactively; `\dt` lists tables, and `EXPLAIN` prints the binder, planner and optimizer stages (`EXPLAIN (o,s)` shows the optimized plan with schemas).
- The course hosts a [BusTub web shell](https://15445.courses.cs.cmu.edu/fall2025/bustub/), a complete reference build running in the browser, which helps settle questions about expected behavior.
- `b_plus_tree_printer` inserts and deletes keys interactively and prints the tree, or writes it to a Graphviz `.dot` file.
- `bustub-sqllogictest` runs a single `.slt` file, and `\dbgmvcc <table>` in the shell calls `TxnMgrDbg`, a debug hook students write to print every tuple's version chain.
- `bustub-bpm-bench`, `bustub-btree-bench` and `bustub-terrier-bench` are the leaderboard workloads for Projects 1, 2 and 4.
- Debug builds enable AddressSanitizer, and `-DBUSTUB_SANITIZER=thread` switches to ThreadSanitizer.

## Ideas that connect the layers {#takeaways}

Reading the system as a whole, a few decisions explain most of the others:

- **Page ids are the only pointers between layers.** B+tree children, the table heap's page links and sort runs all name pages, never memory, so any of them can be evicted and read back.
- **RIDs never change.** That is why a delete only sets a flag, why MVCC updates tuples in place and keeps history elsewhere, and why index entries can outlive deletions.
- **Schemas travel next to data, not inside it.** Plan nodes carry output schemas, and tuples are bytes.
- **Latches protect memory; locks and timestamps protect transactions.** The buffer pool and the B+tree hold latches only for the length of one operation, while MVCC decides what a transaction may see over its whole life.
- **Plan nodes are immutable.** Optimizer rules rebuild trees bottom-up instead of editing them, which keeps rules independent of one another.
