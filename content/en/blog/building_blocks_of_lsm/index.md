---
author: "Sarthak Makhija"
title: "Building blocks of LSM based key/value storage engines: Introduction"
date: 2024-10-10
description: "asdasdasdsadas"
tags: ["LSM", "Building blocks of LSM"]
thumbnail: /bloomfilter_title.webp
caption: ""
---

### LSM-tree: Overview

LSM-tree is a write-optimized data structure implemented by storage engines for supporting write-heavy workloads. A lot of storage 
engines including [BadgerDB](https://github.com/dgraph-io/badger), [RocksDB](https://github.com/facebook/rocksdb) and [LevelDB](https://github.com/google/leveldb) use LSM-tree as their core 
data structure.

> Storage engine is a software module that provides data structures for efficient reads and writes. 
> The two most common data structures are B+Tree (read-optimized) and LSM-tree (write-optimized).

Let's look at the structure of LSM-tree to understand why it is write-optimized.

LSM-tree buffers the data in-memory and performs a sequential writes to disk after the in-memory buffer is full. 
The image below highlights the throughput difference between sequential and random writes on an NVMe SSD; the difference 
would be much higher on an HDD.

<figure>
    <img class="align-center" src="/sequential-random-write.png" /> 
    <figcaption class="figcaption">Sequential write throughput > Random write throughput</figcaption>
</figure>

> LSM-tree-based storage engines offer better write throughput by performing sequential writes to disk.

LSM-tree consists of components of exponentially increasing sizes, C0 to Ck. 
C0 is a *RAM resident* component that stores key-value pairs sorted by key and supports efficient writes and reads, 
whereas C1 to Ck are *immutable disk-resident* components sorted by key.
C0 is known as the [Memtable](/en/blog/building_blocks_of_lsm_memtable/) and C1 to Ck are referred as [SSTables](/en/blog/building_blocks_of_lsm_sstable/).

<img class="align-center" src="/lsm-c0-ck.png" />

> **Data structure choice for C0**: The data structure for C0 should maintain the keys in the sorted order (for range queries) and provide efficient reads and writes. This data structure could be a [treemap](https://docs.oracle.com/javase/8/docs/api/java/util/TreeMap.html) or a [red-black tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree) or a [Skiplist](https://en.wikipedia.org/wiki/Skip_list). 
> Of all these data structures, the Skiplist is the one that supports versioned keys, the same key with different versions.

Let's take a look at the overall flow of `put(key: []byte, value: []byte)` and `get(key: []byte)` operations in the LSM-tree.

Every `put(key, value)` in the LSM-tree adds the key-value pair in the C0 component, and after C0 is full, the entire data is flushed to disk. 
LSM-trees treat `delete(key)` as another `put`, which will put the key with a deleted marker.

After C0 is full, a new instance of C0 is created and the entire in-memory data is flushed to disk. All in-memory data consisting of key-value 
pairs is encoded in a byte array and written to disk. [^1] If the C1 component already exists on disk, the buffered content is merged with the contents of C1.

Because all the new writes are kept in-memory, they can get lost in case of a failure. The durability guarantee is ensured by using 
a [WAL](/en/blog/building_blocks_of_lsm_wal/). 
Every `put(key, value)` *appends* the key-value pair to a WAL file and then writes the key-value pair to the C0 component. Appending 
to a WAL file is also a sequential write to disk. If the storage engine supports transactions, then the `put(key, value)` results 
in addition of the key/value pair to a [Batch](https://github.com/SarthakMakhija/go-lsm/blob/main/kv/batch.go).
When a transaction commits, all batched key-value pairs are appended to the WAL and written to the C0 component

Every `get(key)` in the LSM-tree goes through the RAM based component (C0) to disk components from C1 to Ck in the order. 
The `get(key)` operation first queries the C0 component, if the value for the key is not found, the search proceeds to the disk 
resident component C1. This process continues until the value is found or all the disk resident components have been scanned. 
LSM-trees may need multiple reads for a point lookup. Hence, LSM-trees are most useful when inserts are more common than lookups.

This article series will delve into the fundamental building blocks of LSM tree-based storage engines, providing a comprehensive 
understanding of how they work together to achieve high performance and durability.

This series will cover the following building blocks:

1. [Memtable](/en/blog/building_blocks_of_lsm_memtable/)
2. [WAL](/en/blog/building_blocks_of_lsm_wal/)
3. [SSTables and bloom filter](/en/blog/building_blocks_of_lsm_sstable/)
4. [MVCC and transactions]()
5. [Iterators]()
6. [Manifest]()
7. [Compaction]()

The source code that will be referred throughout the series is available [here](https://github.com/SarthakMakhija/go-lsm). 

Let's introduce [Memtable](/en/blog/building_blocks_of_lsm_memtable/), a fixed-size in-memory data structure.

[^1]: This representation does not consider leveled SSTables.