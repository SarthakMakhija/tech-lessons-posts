---
author: "Sarthak Makhija"
title: "Under the Hood: Serialized Snapshot transaction isolation in Golang"
date: 2024-03-15
description: "
"
tags: ["Golang", "Transaction", "Isolation", "Serialized Snapshot Isolation"]
thumbnail: /diving-into-rust.webp
caption: ""
---

Hello ..

### Introduction

We will start by defining a few terms.

- **Transaction**: is an atomic unit of work. A database transaction may involve writes to multiple tables, or multiple writes to a single table or maybe
multiple writes to multiple tables. Consider a Key/Value storage engine instead of database. We can write the pseudo-code for the transaction as:
```go
transaction := NewReadWriteTransaction().Begin()
transaction.Put([]byte("KeyValueStore"), []byte("BadgerDb"))
transaction.Put([]byte("Isolation"), []byte("Serialized Snapshot Isolation"))
transaction.Delete([]byte("DiskType"))
transaction.Commit()
```
The code snippet creates an instance of a pseudo `ReadWriteTransaction`, `puts` a couple of key/value pairs, `deletes` a key and the `commits` the transaction.
By definition, a transaction is atomic which means, all these actions will take place as a unit or none of them will take place.

- **Isolation**: defines the behavior of the system in presence of other concurrent transactions. A system that provides *Serial* transaction isolation
ensures that all the transactions run serially - one after the other. Imagine that you are building a transactional Key/Value storage engine. The kind 
of questions that you will ask while implementing transaction isolation would include:
  - What happens when two transactions write to the same key at the same time?
  - What happens when a transaction attempts to read the value of a key, which is being written to by another transaction?
  - Can a readonly transaction abort?

- **Snapshot Isolation**: the snapshot from which a transaction reads is not affected by concurrently running transactions. To provide snapshot isolation,
databases and Key/Value storage engines maintain multiple versions of the data. Anytime a transaction begins, it  is given a `beginTimestamp` and 
if a transaction commits without any conflict, it is given a `commitTimestamp`. A transaction **txn<sub>(i)</sub>** with a begin timestamp of 
**T<sub>b</sub>(txn<sub>(i)</sub>)** reads the latest version of data (/key) with the commit timestamp **C** **<** **T<sub>b</sub>(txn<sub>(i)</sub>)**.
Two concurrent transactions can still conflict if there is a **write-write** conflict, meaning, two transactions writing to the same key (in a Key/Value storage engine).

- **Serialized Snapshot Isolation**: the snapshot from which a transaction reads is not affected by concurrently running transactions. The core
idea of maintaining multiple versions of the data remains the same. Two concurrent transactions can still conflict if 
there is a **read-write** conflict, 

- **MVCC**: stands for multi-version concurrency control. It is the backbone for implementing transactions with Snapshot or 
Serialized Snapshot Isolation. In a multi-versioned Key/Value storage engine, each key is given a version which is incremented every time a write 
happens in the storage engine. The following visual should help in building a light mind-map of a read/write transaction flow in a Key/Value storage
engine that supports MVCC with either Snapshot or Serialized Snapshot isolation.

<div class="align-center-exclude-width-change">
    <img src="/transaction_flow_mvcc.png" alt="Transaction Flow with MVCC"/>
</div>

The flow can be summarized as:
- Create an instance of `ReadWriteTransaction` and use the instance to perform relevant operations.
- `Commit` the transaction.
- When the transaction is committed, generate `CommitTimestamp`. `CommitTimestamp` serves as the version for all the keys in the MVCC store.
- Update all the keys in the transaction with the generated `CommitTimestamp`.
- Serially apply all the transactions to the state machine of the Key/Value storage engine.

### Understanding Snapshot Isolation

### Understanding Serialized Snapshot Isolation

### Goodbye Anomalies

### Understanding SkipList

A skiplist is a layered data structure where the lowest layer is an ordinary **ordered** linked list. Each higher layer acts as an 
"express lane" for the lists below. Searching proceeds downwards from the top until consecutive elements bracketing the search element are found.

SkipList can be represented with the following structure:

```go
type SkipList struct {
	header *SkiplistNode
	maxLevel  int
}

type SkiplistNode struct {
	key      int
	value    int
	forwards []*SkiplistNode
}
```

The struct `SkipList` contains a pointer to a header node and each `SkiplistNode` contains a key/value pair along with a slice of forward pointers. 
The header node is the starting node for all the read/write operations and all the key/value pairs will be added to its right. 

One way to ensure that all the key/value pairs are added to the right is to create a header node with some sentinel key/value pair when SkipList is 
instantiated. If we are creating a SkipList to store positive keys, then the sentinel key in the header node can be -1. 
Similarly, if we are creating a SkipList to store string keys, then the sentinel value for the key in the header node can be blank.

> We are using keys and values of type `int` only to serve as example. 

<div class="align-center-exclude-width-change">
    <img src="/skiplist.png" alt="Transaction Flow with MVCC"/>
</div>

The above image is the representation of our `SkipList` struct. 

> `10, 100`, `20, 200` represents key,value pair.

Let's say we want to search the key 25. The following would be the approach:

1. Start with the header node and keep moving towards the right until the right key is greater than the search key.
2. Right of the header node, at the highest level is 10 which is less than 25, so move right.
3. Now we are on the node containing the key 10. 
4. Right of the node containing key 10 is 60 (at the highest level), which is greater than 25. So move to the lower level on the 
same node. Keep moving down until the right node contains a key that is less than or equal to the search key.
5. We will get down to the level 1 (index starting from 0) on the node containing key 10. Right of this level contains 20, so move right.
6. Now we are on the node containing the key 20.
7. Right of the node containing key 20 is 30 which is greater than the search key. So move to the lower level on the same node.
8. Right of level 0 on the node containing key 20 is still 30. There are no more levels to go down to. 
9. So, we can conclude that the key 25 is not present in the list.

SkipList is a good data structure to store multiple versions of each key in a Key/Value storage engine. Each key is given a version, which is
usually the `commitTimestamp` in Snapshot and Serialized Snapshot isolation. 
Each node of SkipList now stores an instance of `VersionedKey` that can be represented as:

```go
type VersionedKey struct {
	key     []byte
	version uint64
}

func (versionedKey VersionedKey) compare(other VersionedKey) int {
    comparisonResult := bytes.Compare(versionedKey.getKey(), other.getKey())
    if comparisonResult == 0 {
        thisVersion, otherVersion := versionedKey.getVersion(), other.getVersion()
        if thisVersion == otherVersion {
            return 0
        }
        if thisVersion < otherVersion {
            return -1
        }
        return 1
    }
    return comparisonResult
}
```

The `Get` method in `SkiplistNode` takes a `VersionedKey` which is the actual key along with the `beginTimestamp` of the transaction. The `Get`
uses the algorithm described earlier. The code is available [here](https://github.com/SarthakMakhija/serialized-snapshot-isolation/blob/main/mvcc/SkiplistNode.go). 

### Implementing Serialized Snapshot Isolation in a Key/Value store

#### Understanding MVCC

#### Implementing Get

#### Implementing Put

### References