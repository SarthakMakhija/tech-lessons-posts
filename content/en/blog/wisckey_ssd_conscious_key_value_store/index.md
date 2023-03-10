---
author: "Sarthak Makhija"
title: "WiscKey: Separating Keys from Values in SSD-Conscious Storage"
date: 2023-03-10
description: "LSM-tree (Log structured merge tree) is a data structure typically used when dealing with write-heavy workloads. LSM-trees optimize the write-path by performing sequential writes to disk. WiscKey is a persistent LSM-tree-based key-value store that separates keys from values to minimize read and write amplification. The design of WiscKey is highly SSD optimized, leveraging both the sequential and random performance characteristics of the device."
tags: ["Storage engine", "LSM-tree", "WiscKey", "SSD-conscious"]
thumbnail: /wisckey.jpg
caption: "Background by Alex Conchillos on Pexels"
---

> LSM-tree (Log structured merge tree) is a data structure typically used for write-heavy workloads. LSM-trees optimize the write path by performing sequential writes to disk. WiscKey is a persistent LSM-tree-based key-value store that separates keys from values to minimize read and write amplification. The design of WiscKey is highly SSD optimized, leveraging both the device's sequential and random performance characteristics.

This article summarises the [WiscKey](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf) paper published in 2016.

Before we understand the paper, it is essential to understand the LSM-tree data structure, read and write amplification in the LSM-tree and various SSD features that can be leveraged while building SSD-conscious storage engine.

### LSM-tree

LSM-tree is a write-optimized data structure implemented by storage engines for supporting write-heavy workloads. A lot of storage engines including [BadgerDB](), [RocksDB]() and
[LevelDB]() use LSM-tree as the core data structure.

> Storage engine is a software module that provides data structures for efficient reads and writes. The two most common data structures are B+Tree (read-optimized) and LSM-tree (write-optimized).

Let's look at the structure of LSM-tree to understand why it is write-optimized. LSM-tree consists of components of exponentially increasing sizes, C0 to Ck. C0 is a *RAM resident* component that stores key-value pairs sorted by key and supports efficient writes and reads, whereas C1 to Ck are *immutable disk-resident* components sorted by key.

<img class="align-center" src="/lsm-c0-ck.png" />

> **Data structure choice for C0**: The data structure for C0 should maintain the keys in the sorted order and provide efficient reads and writes. This data structure could be a [treemap](https://docs.oracle.com/javase/8/docs/api/java/util/TreeMap.html) or a [red-black tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree) or a [skip list](https://en.wikipedia.org/wiki/Skip_list). Of all these data structures, the Skip list is the one that supports versioned keys.

LSM-tree buffers the data in-memory and performs a sequential write to disk after the in-memory buffer is full. The below image highlights the throughput difference between sequential and random writes on an NVMe SSD, the difference would be a lot higher on an HDD. 

<figure>
    <img class="align-center" src="/sequential-random-write.png" /> 
    <figcaption class="figcaption">Sequential write throughput > Random write throughput</figcaption>
</figure>

> LSM-tree based storage engines offer better write throughput because they perform sequential writes to disk.

Let's take a look at the overall flow of `put(key, value)` and `get(key)` operations in an LSM-tree.

Every `put(key, value)` in the LSM-tree adds the key-value pair in the C0 component and after C0 is full, the entire data is flushed to disk. LSM-trees treat `delete(key)` as another `put` which will put the key with a deleted marker. 

After C0 is full, the entire data is flushed to disk. Simply speaking, the entire in-memory data consisting of key-value pairs is encoded in a byte array and written to disk. If the C1 component already exists on disk,   
the buffered content is merged with the contents of C1. **More on this later. <<<<<<<<<<<<<<<<<<<>>>>>>>>>>>>>**

> To ensure persistent writes, every `put(key, value)` *appends* the key-value pair to a [WAL](https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html) file and then writes the key-value pair to the C0 component. Appending to a WAL file is also a sequential write to disk.   

Every `get(key)` in the LSM-tree goes through the RAM based component to disk components from C1 to Ck in the order. The `get(key)` operation first queries the C0 component, if the value for the key is not found, the search proceeds
to the disk resident component C1. This process continues until the value is found or all the disk resident components have been scanned. LSM-trees may need multiple reads for a point lookup. Hence, LSM-trees are most useful when inserts are more common than lookups. 

Let's look at the structure of LSM-tree in LevelDB. 

### LevelDB

LevelDB is a key-value storage engine based on LSM-tree. LevelDB maintains the following data structures:
1. On-disk log file to persist the writes
2. Two in-memory components called "memtable" (active and passive)
3. Seven levels of on-disk components called "SSTable" (Sorted string table) 

> LevelDB implements the in-memory components using Skip list.

Every `put(key, value)` goes in a log file and the active in-memory memtable. Once the active memtable is full, LevelDB *switches to a new memtable and log file* to handle further writes.

The previously active memtable is stored as an immutable memtable in RAM and flushed to disk in the background. This flush to disk generates a new SSTable file (about 2 MB) at level-0 and also discards the previous log file.

The size of all files in each level is limited, and increases by a factor of ten with the level number. For example, the size limit of all files at L1 is 10 MB, while the limit of L2 is 100 MB.

In order to perform the `get(key)` operation, LevelDB performs the following steps:

1. Perform `get` in the active memtable (RAM)
2. If not found, perform `get` in the immutable memtable (RAM)
3. If not found, perform `get` in the files from Level0 to Level6. LevelDB ensures that the keys do not overlap in the files from Level1 to Level6 whereas keys in the Level0 files can overlap.
   1. This means that a `get` operation may involve multiple files in Level0 and one file at each level from Level1 to Level6

The below image represents the high-level architecture of LevelDB.

<img class="align-center" src="/leveldb.png" />

> [BadgerDB](https://github.com/dgraph-io/badger) maintains an array of immutable memtables. The `get` operation searches the active memtable and if the key is not found, it searches the inactive (or the immutable) memtables in the reverse order (the newest immutable memtable to the oldest immutable memtable).   

<<<<Compaction>>>

### Read-Write amplification

### SSD features

### WiscKey proposal

### Challenges with WiscKey

### Implementation of WiscKey

### References

- [WiscKey](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)
- [LSM tree](https://segmentfault.com/a/1190000041198407/en)
