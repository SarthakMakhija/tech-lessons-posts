---
author: "Sarthak Makhija"
title: "Building blocks of LSM based key/value storage engines: Memtable"
date: 2024-10-10
description: "asdasdasdsadas"
tags: ["LSM", "Building blocks of LSM", "Memtable"]
thumbnail: /bloomfilter_title.webp
caption: ""
---

### Memtable

A fixed-size in-memory data structure that temporarily stores incoming writes until it reaches capacity. Each memtable typically 
has a corresponding Write-Ahead Log (WAL) to ensure durability.

Storage engines like Badger maintain an active (or current) memtable and a collection of inactive (or immutable) memtables. 
When the active memtable fills up, its WAL is flushed, and the memtable becomes immutable. A new active memtable is then created 
to handle subsequent writes.

Writing a key-value pair to an LSM-based storage engine involves:

- **Capacity Check**: Determine if the active memtable has enough space.
- **Write to memtable**: If there's space, append the key-value pair to both the active memtable and its associated WAL.
- **Memtable flush**: If the memtable is full, flush its WAL, convert it to an immutable memtable, and create a new active memtable. 
Then, write the key-value pair to the new active memtable and its WAL.

This write-flow is a simplified overview, and it will evolve as our understanding of the building blocks of an LSM-based storage engine
improves.

### Memtable data structure requirements and options

The choice of data structure for the memtable in an LSM tree storage engine hinges on three key requirements:

1. **Efficient reads and writes**: Ideally, the data structure should support both reads and writes with an average logarithmic time complexity of log₂N, ensuring fast insertions and retrievals.
2. **Range queries**: The ability to efficiently perform range queries on keys is crucial for certain use cases. This allows the retrieval of a set of key-value pairs within a specific key range.
3. **Optional, multiple key versions (for transactions)**: While not a strict requirement, some implementations supporting transaction isolation levels like snapshot or serialized snapshot need to maintain multiple versions of the same key. In such cases, the data structure should keep all versions of a key together for efficient access.

Here are some options:

- **TreeMap**: This offers O(log N) for reads and writes but doesn't inherently support multiple versions of the same key.
- **Red-Black Tree**: Similar to **TreeMap**, it provides efficient reads and writes but lacks built-in support for versioning.
- **Skiplist**: This data structure offers logarithmic time complexity and can be easily adapted to store multiple versions of a key
efficiently, making it a good choice for memtable.  

### Skiplist

A Skiplist is a layered data structure where the lowest layer is an ordered linked list. Each higher layer acts as an “express lane” for the lists below. Searching proceeds downwards from the top until consecutive elements bracketing the search element are found.

An example `Skiplist` can be represented with the following structure:

```go
type Skiplist struct {
	header *SkiplistNode
	maxLevel  int
}

type SkiplistNode struct {
	key      int
	value    int
	forwards []*SkiplistNode
}
```

The struct `Skiplist` contains a pointer to a header node and each `SkiplistNode` contains a key/value pair along with a slice of forward pointers. 
The header node is the starting node for all the read/write operations and all the key/value pairs will be added to its right.

One way to ensure that all the key/value pairs are added to the right is to create a header node with some sentinel key/value pair 
when `Skiplist` is instantiated. If we are creating a `Skiplist` to store positive keys, then the sentinel key in the header node 
can be -1. Similarly, if we are creating a `Skiplist` to store string keys, then the sentinel value for the key in the header node 
can be blank.

> We are using keys and values of type int only to serve as example.

<div class="align-center-exclude-width-change">
    <img src="/Skiplist.png" alt="Transaction Flow with MVCC"/>
</div>

The above image is the representation of our `Skiplist` struct.

> `(10, 100)`, `(20, 200)` represent key,value pairs.

Let's say we want to search the key 25. The following will be the approach:

1. Start with the header node at the highest level and keep moving towards the right until the right key is greater than the search key.
2. Right of the header node, at the highest level is 10 which is less than 25, so move right.
3. Now we are on the node containing the key 10.
4. Right of the node containing key 10 is 60 (at the highest level), which is greater than 25. So move to the lower level on the
   same node. Keep moving down until the right node contains a key that is less than or equal to the search key.
5. We will get down to the level 1 (index starting from 0) on the node containing key 10. Right of this level contains 20, so move right.
6. Now we are on the node containing the key 20.
7. Right of the node containing key 20 is 30 which is greater than the search key. So move to the lower level on the same node.
8. Right of level 0 on the node containing key 20 is still 30. There are no more levels to go down to.
9. So, we can conclude that the key 25 is not present in the list.

Above is a simplified example of a Skiplist where keys are integers, but in an LSM-based storage engine with multiple key 
versions, the [key](https://github.com/SarthakMakhija/go-lsm/blob/main/kv/key.go) structure would look like the following:

```go
// Key represents a versioned key. It contains the original key (of the raw key) along with 
// the timestamp.
// A small comment on timestamp:
// When an instance of Key is stored in the system, the timestamp is the commit-timestamp, 
// generated by txn.Oracle.
// When the user performs a get (/read) operation, the system creates an instance of Key with the 
// user provided key and the begin-timestamp of the transaction.
type Key struct {
	key       []byte
	timestamp uint64
}
```

### Ordering multi-versioned keys in Skiplist

When using Skiplist with versioned keys (keys associated with timestamps), it's crucial to determine the order in which 
keys with the same name but different timestamps should be arranged. 
A common approach is to prioritize keys with higher timestamps over those with lower timestamps. This ordering simplifies 
range iterator.

Consider a system where the key "consensus" has two versions: one with a timestamp of 9 and another with a timestamp of 7. 
Now, imagine a scan operation that seeks to retrieve all keys between "consensus" and "etcd". 
If the user's transaction has a begin-timestamp of 10, we only want to return keys whose commit-timestamps are less than 10.

By placing the key "consensus" with the timestamp 9 before the one with timestamp 7, the range iterator can easily 
identify and return the latest committed version of the key (the one with timestamp 9). It can efficiently skip older 
versions of the same key, ensuring that only the most recent relevant data is returned.

This prioritization of higher-timestamp keys significantly simplifies the implementation of range iterators in Skiplist-based 
storage systems.

[Key comparison](https://github.com/SarthakMakhija/go-lsm/blob/main/kv/key.go#L63) typically looks like the following:

```go
func (key Key) CompareKeysWithDescendingTimestamp(other Key) int {
	comparison := bytes.Compare(key.key, other.key)
	if comparison != 0 {
		return comparison
	}
	if key.timestamp == other.timestamp {
		return 0
	}
	if key.timestamp > other.timestamp {
		return -1
	}
	return 1
}
```

Let's look at the concurrent implementation choices for Skiplist.

### Lock-based or lock-free

When using a Skiplist as the underlying data structure for the memtable, the choice between lock-based and lock-free 
implementations is crucial for concurrent operations. 

Lock-based approaches, while simpler to implement, can introduce contention and limit concurrency. This is particularly 
problematic for `scan` operations, which require acquiring read lock either on the entire skiplist or on multiple skiplist nodes. 
If the memtable supports `scan` operations, a lock-based implementation can significantly hinder write performance as no writes 
can be performed while a scan is in progress. 

To address this, lock-free algorithms offer a more scalable and concurrent solution, allowing multiple threads to modify 
the Skiplist simultaneously without relying on locks.

[Pebble](https://github.com/cockroachdb/pebble/blob/999b65aeb0525a761bb4b78311945a9995769ff1/internal/arenaskl/skl.go)
and [Badger](https://github.com/dgraph-io/badger/blob/main/skl/skl.go) provide concurrent and lock-free implementation of Skiplist.

### Iterator

Many memtable implementations provide [iterators](https://github.com/SarthakMakhija/go-lsm/blob/main/memory/memtable.go#L177) to facilitate various operations:

1. **Scan Operations**: Iterators enable [scanning through the memtable](https://github.com/SarthakMakhija/go-lsm/blob/main/memory/memtable.go#L117), 
iterating from the starting key to the ending key. This is useful for range queries.
2. **Flushing to SSTable**: Iterators are also useful in the process of [flushing](https://github.com/SarthakMakhija/go-lsm/blob/main/state/storage_state.go#L288) the memtable to an SSTable (Sorted String Tables). 
By iterating over the [entries of the memtable](https://github.com/SarthakMakhija/go-lsm/blob/main/memory/memtable.go#L123), the iterator can efficiently extract the data and write it to the SSTable.

### Summary

A memtable is a fixed-size in-memory data structure that acts as a temporary buffer for incoming writes in LSM-based storage 
engines. Each memtable typically has a corresponding Write-Ahead Log (WAL) for durability.
When the memtable reaches capacity, it becomes immutable and a new active memtable is created.

Skiplist is a good choice for implementing memtable because they provide efficient retrieval and have the ability to store 
multiple versions of a key together. Skiplist can be implemented with either lock-based or lock-free concurrency control. 
Lock-based implementations are simpler but can limit concurrency, especially for scan operations. Lock-free implementations 
offer better scalability for concurrent operations.

Many memtable implementations provide iterators to facilitate tasks like scanning through the memtable and flushing entries to SSTables (Sorted String Tables).

### Code

Memtable implementation is available [here](https://github.com/SarthakMakhija/go-lsm/blob/main/memory/memtable.go). This 
implementation leverages a lock-free Skiplist implementation from [Badger](https://github.com/dgraph-io/badger/blob/main/skl/skl.go) with [minor modifications](https://github.com/SarthakMakhija/go-lsm/tree/main/memory/external).

Let's introduce durability by using [WAL](/en/blog/building_blocks_of_lsm_wal/).