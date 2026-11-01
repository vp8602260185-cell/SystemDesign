# B-Trees and LSM Trees: Study Notes

---

# Part 1: B-Trees

## Why they exist

Binary search trees are fast in memory, but on disk each node visit is a slow read. A balanced binary tree with a million keys is about 20 levels deep, which means about 20 disk reads per lookup.

B-trees fix this by making each node **wide**: one node holds many keys and is sized to fit a disk page (often 4-16 KB). With hundreds of children per node, a tree holding millions of keys is only 3-4 levels deep, so a lookup costs 3-4 disk reads.

## The rules

For a B-tree of order *m* (max children per node):

1. Each node holds up to *m − 1* keys, stored in sorted order.
2. A node with *k* keys has *k + 1* children. The keys act as dividers: everything in child *i* falls between key *i−1* and key *i*.
3. Every non-root node is at least half full (at least ⌈m/2⌉ children).
4. **All leaves are at the same depth.** This is what keeps the tree balanced.

Example, order 4 (max 3 keys per node):

```
              [ 20 | 40 ]
             /     |     \
    [5|10|15]  [25|30|35]  [45|50|55]
```

## Search

1. Start at the root.
2. Find the first key >= target (scan or binary search within the node).
3. If it matches, done. Otherwise follow the child pointer to the left of that key.
4. Repeat until a match or a leaf.

Example: to find 30, at the root 30 is between 20 and 40, so go to the middle child and find it.

Cost: O(log n) node visits.

## Insert

1. Search down to the correct leaf.
2. Insert the key in sorted position.
3. If the leaf has too many keys, **split** it: the median key moves up to the parent, and the remaining keys form two nodes.
4. If the parent overflows, split it too. This can cascade to the root. If the root splits, a new root is created and the tree grows one level taller.

Example: inserting 12 into `[5|10|15]` gives `[5|10|12|15]` (too big). Promote 12 to the parent, leaving `[5|10]` and `[15]`. The root becomes `[12|20|40]`.

> The tree grows **upward**, not downward, which is why all leaves stay at the same depth.

## Delete

1. Remove the key. If it is in an internal node, swap it with its in-order predecessor or successor from a leaf first.
2. If the node is too empty, **borrow** a key from a sibling that has extras (rotate through the parent).
3. If no sibling can spare one, **merge** with a sibling and pull the divider key down from the parent. This can cascade upward.

## B+ trees

Most real databases use this variant:

- Internal nodes store only routing keys. All data lives in the leaves.
- Leaves are linked in a list, so range queries (e.g. "all orders between March and May") are a fast sequential scan.
- Internal nodes are smaller, so more keys fit per page and the tree is even shallower.

## Where they're used

- **Databases:** PostgreSQL, MySQL (InnoDB), SQLite, Oracle (B+ tree indexes)
- **File systems:** NTFS, ext4, Btrfs, APFS (B-tree variants for metadata)

## Complexity

Search, insert, delete: all O(log n), but the log base is the fanout (large), so the practical constant is tiny.

---

# Part 2: LSM Trees (Log-Structured Merge Trees)

## The core idea

B-trees update data **in place**, which means random disk writes. Random writes are slow, especially on spinning disks, and still costly on SSDs because of write amplification.

LSM trees flip this: **never modify data on disk, only append.** Writes go to memory first, get flushed to disk in big sorted batches, and are cleaned up later in the background. This turns random writes into sequential ones.

## Components

```
  Write ──► [WAL]  (append-only log on disk, for crash recovery)
        └─► [Memtable]  (sorted structure in RAM)
                │ when full, flush
                ▼
   Level 0:  [SST] [SST] [SST]        (newest, may overlap)
   Level 1:  [SST] [SST] [SST] [SST]  (sorted, non-overlapping)
   Level 2:  [SST] [SST] ... (10x bigger)
```

- **WAL (write-ahead log):** every write is appended here first so the memtable can be rebuilt after a crash.
- **Memtable:** in-memory sorted structure (skip list or red-black tree) that absorbs writes.
- **SSTable (Sorted String Table):** immutable sorted file on disk, created when a memtable is flushed. Never modified after it is written.

## Write path

1. Append to the WAL.
2. Insert into the memtable.
3. When the memtable is full, write it out as a new SSTable.

No disk seeks, no page splits, so writes are very fast.

## Read path

A key could be in several places:

1. Check the memtable.
2. Check SSTables from newest to oldest. The first match wins, since newer data shadows older data.

Two tricks avoid checking every file:

- **Bloom filters:** a small in-memory filter per SSTable that says "definitely not here" or "maybe here." Skips most files.
- **Sparse indexes:** within a file, an index narrows the lookup to one block.

## Deletes

Immutable files can't have data removed, so a delete writes a **tombstone** (a "this key is deleted" marker). Reads that hit a tombstone return "not found." The real data is dropped later during compaction.

## Compaction

Over time SSTables pile up with old versions and tombstones. A background process **merges** files (like the merge step in merge sort):

- Keeps only the newest version of each key
- Drops tombstones and overwritten values
- Writes out new, larger sorted files

| Strategy | How it works | Favors |
|---|---|---|
| **Leveled** (RocksDB, LevelDB default) | Each level ~10x larger, non-overlapping files | Reads and space efficiency |
| **Size-tiered** (Cassandra default) | Merge similar-sized files together | Write throughput |

## The three amplifications

- **Write amplification:** one logical write is rewritten many times by compaction
- **Read amplification:** a read may check multiple files
- **Space amplification:** stale data lingers until compaction

LSM design is a tradeoff among these; you can tune toward one at the expense of the others.

## Where they're used

LevelDB, RocksDB, Cassandra, HBase, ScyllaDB, InfluxDB. RocksDB is also embedded in other systems such as CockroachDB and TiKV.

---

# Part 3: B-Tree vs. LSM Tree

| | B-tree | LSM tree |
|---|---|---|
| Writes | Random, in place | Sequential, append-only |
| Write speed | Moderate | Very high |
| Read speed | Fast, predictable | Slower, variable |
| Background work | Little | Compaction (can cause latency spikes) |
| Best for | Read-heavy, point lookups | Write-heavy, ingestion workloads |

---

# My Notes / Questions

-