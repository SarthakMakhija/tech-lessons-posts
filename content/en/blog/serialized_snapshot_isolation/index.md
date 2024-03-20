---
author: "Sarthak Makhija"
title: "A guide to Serializable Snapshot Isolation in Key/Value storage engine"
date: 2024-03-15
description: "
Ensuring data consistency in the face of concurrent transactions is a critical challenge in database management. 
This article explores Serializable Snapshot Isolation (SSI), a rising star in the field that promises the best of both worlds: 
strong data consistency without sacrificing performance. 
The article delves into the inner workings of SSI and explore its implementation for a Key/Value storage engine.
"
tags: ["Golang", "Transaction", "Isolation", "Serializable Snapshot Isolation", "BadgerDb"]
thumbnail: /serializable-snapshot-isolation.png
caption: "Background by Lum3n on Pexels"
---

Ensuring data consistency in the face of concurrent transactions is a critical challenge in database management. 
Traditional serializable isolation, while guaranteeing data integrity, often suffers from performance bottlenecks due to extensive locking. 
This article explores Serializable Snapshot Isolation (SSI), a rising star in the field that promises the best of both worlds: 
strong data consistency without sacrificing performance. 
The article delves into the inner workings of SSI and explore its implementation for a Key/Value storage engine. We will refer to the research
paper titled [A critique of snapshot isolation](https://dl.acm.org/doi/10.1145/2168836.2168853).

### Introduction

We will start by defining a few terms.

- **Transaction**: is an atomic unit of work. A database transaction may involve writes to multiple tables, or multiple writes to a single table or maybe
multiple writes to multiple tables. Consider a Key/Value storage engine instead of database. We can write the pseudo-code for the transaction as:
```go
transaction := NewReadWriteTransaction().Begin()
transaction.Put([]byte("KeyValueStore"), []byte("BadgerDb"))
transaction.Put([]byte("StoreType"), []byte("LSM"))
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

- **Snapshot isolation**: the snapshot from which a transaction reads is not affected by concurrently running transactions. To support snapshot isolation,
databases and Key/Value storage engines maintain multiple versions of the data. Anytime a transaction begins, it  is given a `beginTimestamp` and 
if a transaction commits without any conflict, it is given a `commitTimestamp`. A transaction **txn<sub>(i)</sub>** with a begin timestamp of 
**T<sub>b</sub>(txn<sub>(i)</sub>)** reads the latest version of data (/key) with the commit timestamp **C** **<** **T<sub>b</sub>(txn<sub>(i)</sub>)**.
Two concurrent transactions can still conflict if there is a **write-write** conflict, meaning, two transactions writing to the same key (in a Key/Value storage engine).

- **Serializable Snapshot isolation**: the snapshot from which a transaction reads is not affected by concurrently running transactions. The core
idea of maintaining multiple versions of the data remains the same. Two concurrent transactions can still conflict if 
there is a **read-write** conflict, 

- **MVCC**: stands for multi-version concurrency control. It is the backbone for implementing transactions with Snapshot or 
Serializable Snapshot isolation. In a multi-versioned Key/Value storage engine, each key is given a version which is incremented every time a write 
happens in the storage engine. The following visual should help in building a light mind-map of a read/write transaction flow in a Key/Value storage
engine that supports MVCC with either Snapshot or Serializable Snapshot isolation.

<div class="align-center-exclude-width-change">
    <img src="/transaction_flow_mvcc.png" alt="Transaction Flow with MVCC"/>
</div>

The flow can be summarized as:
- Create an instance of `ReadWriteTransaction` and use the instance to perform relevant operations.
- `Commit` the transaction.
- When the transaction is committed, generate `CommitTimestamp`. (`CommitTimestamp` serves as the version for all the keys in the MVCC store).
- Update all the keys in the transaction with the generated `CommitTimestamp`.
- Serially apply all the transactions to the state machine of the Key/Value storage engine.

### Understanding Snapshot isolation

In **Snapshot isolation**, the snapshot from which a transaction reads is not affected by concurrently running transactions. To support snapshot isolation,
databases and Key/Value storage engines maintain [multiple versions of the data](#skiplist-and-mvcc). 

The high-level implementation of `Get` in **Snapshot isolation**, in a Key/Value storage engine can be summarized as:
- Each transaction is given a `beginTimestamp`. It is an increasing number.
- Timestamps are assigned by a centralized authority called `Oracle` (or `Timestamp Oracle`).
- A transaction **txn<sub>(i)</sub>** with a begin timestamp of **T<sub>b</sub>(txn<sub>(i)</sub>)** reads the latest version of keys 
with the commit timestamp **C** **<** **T<sub>b</sub>(txn<sub>(i)</sub>)**.
- Simply put, a transaction **txn<sub>(i)</sub>** observes all the changes that have been committed before the start of **txn<sub>(i)</sub>**.

Before we explore the intricacies of `Oracle` implementation, let's briefly examine the `beginTimestamp` method.

```go
type Oracle struct {
    lock            sync.Mutex
    nextTimestamp   uint64
	//other fields omitted
}

func NewOracle(transactionExecutor *TransactionExecutor) *Oracle {
  oracle := &Oracle{
    nextTimestamp: 1,
  }
  return oracle
}

func (oracle *Oracle) beginTimestamp() uint64 {
    oracle.lock.Lock()
    beginTimestamp := oracle.nextTimestamp - 1
    oracle.lock.Unlock()
    
	//code omitted
	return beginTimestamp
}
```

Every transaction gets a `beginTimestamp` which is represented as `uint64` (64 bits).
The `nextTimestamp` field of `Oracle` denotes the `commitTimestamp` that shall be given to the next transaction that is ready to commit. Based on the field `nextTimestamp`, we can derive the `beginTimestamp`.

- If the `nextTimestamp` is 10, the next transaction that is ready to commit will be given 10 as its `commitTimestamp`.
- This means that 9 is the latest timestamp that is given to some transaction **txn<sub>(some)</sub>** as its `commitTimestamp`.
- This means that the current transaction **txn<sub>(current)</sub>** can be awarded 9 as its `beginTimestamp`. 
- So, `beginTimestamp = nextTimestamp - 1`.
- Simply put, **txn<sub>(current)</sub>** can read keys with `commitTimestamp` < 9, where 9 is the `beginTimestamp` of **txn<sub>(current)</sub>**.

A `ReadWriteTransaction` is assigned a `commitTimestamp` when it is ready to commit. Before assigning the `commitTimestamp`, it is necessary to 
check that there are no *write-write* conflicts. Two concurrent transactions can conflict in Snapshot isolation if:
1. Both the transactions **txn<sub>(i)</sub>** and **txn<sub>(j)</sub>** write to the same key called *Spatial overlap*, and
2. **T<sub>b</sub>(txn<sub>(i)</sub>)** `<` **T<sub>c</sub>(txn<sub>(j)</sub>)** and **T<sub>b</sub>(txn<sub>(j)</sub>)** `<` **T<sub>c</sub>(txn<sub>(i)</sub>)**, 
where **T<sub>b</sub>(txn<sub>(i)</sub>)** represents the `beginTimestamp` of **txn<sub>(i)</sub>** and **T<sub>c</sub>(txn<sub>(j)</sub>)** represents
the `commitTimestamp` of **txn<sub>(j)</sub>**. This is called *Temporal overlap*.

Let's write the pseudo-code for committing a transaction **txn<sub>(i)</sub>** with `beginTimestamp` as **T<sub>b</sub>**.

```shell
## detect conflict
for each key in keysToCommit:
    if lastCommit(key) > txn.beginTimestamp()
	    abort;
    end if;
end for;

## prepare for commit
commitTimestamp <- oracle.commitTimestamp();
for each key in keysToCommit:
   lastCommit(key) <- commitTimestamp
end for;

## apply commits
```

The pseudo-code checks for:
- spatial overlap, and
- only the first part of temporal overlap condition. If a newer commit exists for any key that the transaction is going to write such that, `lastCommit(key) > txn.beginTimestamp()`, 
the transaction is aborted. This ensures that the transaction only operates on data that reflects the state at the start of the transaction, 
preventing inconsistencies caused by concurrent modifications.

#### Anomalies

<div class="align-center-exclude-width-change">
    <img src="/snapshot-isolation-anomalies.png" alt="Anomalies prevented by Snapshot isolation"/>
    <figcaption class="figcaption text-sm">Snapshot Isolation prevents many anomalies but not write skew. Question mark image from <a href="https://pixabay.com/vectors/question-mark-thinking-question-5656992/">Pixabay</a></figcaption>
</div>

#### Write skew

Two concurrent transactions can conflict in Snapshot isolation if they write to the same key and there is a temporal overlap. 
But, what if two transactions write to different keys which are related by some constraint. 

Consider the following:
- two keys `x` and `y` with initial values as 1 and related by the constraint `x + y > 0`.
- two concurrent transactions, **txn<sub>(x)</sub>** and **txn<sub>(y)</sub>**, both of which get a `beginTimestamp` of 1, read the
values of `x` and `y` and get 1 as the value for each key. 
- **txn<sub>(x)</sub>** reduces the value of `x` by 1 and **txn<sub>(y)</sub>** reduces the value of `y` by 1.

The following events highlight an anomaly called *write skew*.

- The transaction **txn<sub>(x)</sub>** gets a `commitTimestamp` of 2, checks that `x + y > 0`, reduces the value of `x` by 1 and 
writes the versioned value of `x` back to the store. At version 2, `x` becomes 0.
- The transaction **txn<sub>(y)</sub>** gets a `commitTimestamp` of 3, checks that `x + y > 0`, [[it will read the values of x and y using its `beginTimestamp`]] 
reduces the value of `y` by 1 and writes the versioned value of `y` back to the store. At version 3, `y` becomes 0.
- Another readonly transaction **txn<sub>(another)</sub>** starts at a later point in time, gets a `beginTimestamp` of 6 and reads the values of `x` and `y`:
  - reads the latest value of `x` such that the `commitTimestamp` of `x` `<` 6. It gets `x = 0`. [`x` became 0 at version 2]  
  - reads the latest value of `y` such that the `commitTimestamp` of `y` `<` 6. It gets `y = 0`. [`y` became 0 at version 3]

The constraint `x + y > 0` is broken. Snapshot isolation does not prevent write skew.

> Storage systems like Percolator, [Dgraph](https://github.com/dgraph-io/dgraph) implement Snapshot isolation.

### Understanding Serializable Snapshot isolation

Serializable Snapshot isolation derives the ideas of `beginTimestamp`, `commitTimestamp` and `Oracle` from Snapshot isolation. The difference is the 
way conflict is detected between two concurrently running transactions.

A transaction **txn<sub>(j)</sub>** conflicts with another transaction **txn<sub>(i)</sub>**, if:
1. **txn<sub>(j)</sub>** writes to the keys read by **txn<sub>(i)</sub>**, and
2. **txn<sub>(j)</sub>** commits during the lifetime of **txn<sub>(i)</sub>**, **T<sub>b</sub>(txn<sub>(i)</sub>)** `<` **T<sub>c</sub>(txn<sub>(j)</sub>)** < **T<sub>c</sub>(txn<sub>(i)</sub>)**.

Here, **T<sub>b</sub>(txn<sub>(i)</sub>)** represents the `beginTimestamp` of the transaction **txn<sub>(i)</sub>** and 
**T<sub>c</sub>(txn<sub>(i)</sub>)** represents the `commitTimestamp` of the transaction **txn<sub>(i)</sub>**.

Let's write the pseudo-code for committing a transaction **txn<sub>(i)</sub>** with `beginTimestamp` as **T<sub>b</sub>**.

```shell
## detect conflict
for each key in keysRead:
    if lastCommit(key) > txn.beginTimestamp()
	    abort;
    end if;
end for;

## prepare for commit
commitTimestamp <- oracle.commitTimestamp();
for each key in keysToCommit:
   lastCommit(key) <- commitTimestamp
end for;

## apply commits
```

The pseudo-code checks to see that none of the keys read by the transaction **txn<sub>(i)</sub>** have been committed after it began.
The implementation of Serializable Snapshot isolation will require each read-write transaction to keep track of the read keys. 

> [BadgerDb](https://github.com/dgraph-io/badger) implements Serializable Snapshot isolation. 

Before diving into the implementation of Serializable Snapshot isolation, let's quickly take a look at SkipList and MVCC.

### SkipList and MVCC

A SkipList is a layered data structure where the lowest layer is an ordinary **ordered** linked list. Each higher layer acts as an 
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

> `(10, 100)`, `(20, 200)` represent key,value pairs.

Let's say we want to search the key 25. The following will be the approach:

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

SkipList is a good data structure in a sense that it allows us to store multiple versions of each key (in a Key/Value storage engine). 
Each key is given a version, which is usually the `commitTimestamp` in Snapshot and Serializable Snapshot isolation. 
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

The `Get` method in `SkiplistNode` takes a `VersionedKey`, that combines the target key the user wants to search along with the `beginTimestamp` of the transaction. The method `Get`
uses the algorithm described earlier. The code is available [here](https://github.com/SarthakMakhija/serialized-snapshot-isolation/blob/main/mvcc/SkiplistNode.go). 

We have covered all the building blocks of Serialized Snapshot isolation. It is time to implement it.

### Implementing Serializable Snapshot isolation in a Key/Value store

This article describes an [implementation](https://github.com/SarthakMakhija/serialized-snapshot-isolation) of Serialized Snapshot isolation 
that was implemented with [BadgerDb](https://github.com/dgraph-io/badger/) as reference. This implementation focuses specifically on the *isolation* property of transactions.

#### Introduction

Let me start by introducing core concepts in the code.

1. **ReadonlyTransaction**: provides support for `Get` method. A `ReadonlyTransaction` never aborts.
2. **ReadWriteTransaction**: provides support for `PutOrUpdate` and `Get` methods. A `ReadWriteTransaction` can abort if there is a Read-Write conflict with another transaction.
3. **Oracle**: awards `beginTimestamp` and `commitTimestamp` to transactions. It also checks for conflicts between `ReadWriteTransaction`(s).
4. **TransactionExecutor**: applies transactions serially, one after the other.
5. **TransactionTimestampMark**: keeps track of the timestamps that are processed. These could be `beginTimestamp` or `commitTimestamp`.
6. **MemTable**: an in-memory data structure that uses [SkipList](#skiplist-and-mvcc) for storing versioned keys. It acts as the state machine.

Let's start by implementing `ReadonlyTransaction`.

#### Implementing ReadonlyTransaction

`ReadonlyTransaction` provides `Get` method that fetches the value for the given key from the state machine.

Let's take a look at the definition or `ReadonlyTransaction`.

```go
type ReadonlyTransaction struct {
	beginTimestamp uint64
	memtable       *mvcc.MemTable
	oracle         *Oracle
}
```
A `ReadonlyTransaction` has a `beginTimestamp` which is obtained from `Oracle` and an instance of `MemTable` to fetch the value for a key.

Anytime a new instance of `ReadonlyTransaction` is created, we get the `beginTimestamp` from `Oracle`. 

```go
func NewReadonlyTransaction(oracle *Oracle) *ReadonlyTransaction {
  return &ReadonlyTransaction{
    beginTimestamp: oracle.beginTimestamp(),
    oracle:         oracle,
    memtable:       oracle.transactionExecutor.memtable,
  }
}
```

We have already see the implementation of `beginTimetamp` method of `Oracle` earlier, but we can take a quick look again.

```go
func (oracle *Oracle) beginTimestamp() uint64 {
	oracle.lock.Lock()
	beginTimestamp := oracle.nextTimestamp - 1
	oracle.beginTimestampMark.Begin(beginTimestamp)
	oracle.lock.Unlock()

	_ = oracle.commitTimestampMark.WaitForMark(context.Background(), beginTimestamp)
	return beginTimestamp
}
```

The `nextTimestamp` field of `Oracle` denotes the `commitTimestamp` that shall be given to the next transaction that is ready to commit. 
Based on the field `nextTimestamp`, we can derive the `beginTimestamp`.

- If the `nextTimestamp` is 10, the next transaction that is ready to commit will be given 10 as its `commitTimestamp`.
- This means that 9 is the latest timestamp that is given to some transaction **txn<sub>(some)</sub>** as its `commitTimestamp`.
- This means that the current transaction **txn<sub>(current)</sub>** can be awarded 9 as its `beginTimestamp`.
- So, `beginTimestamp = nextTimestamp - 1`.
- Simply put, **txn<sub>(current)</sub>** can read keys with `commitTimestamp` < 9, where 9 is the `beginTimestamp` of **txn<sub>(current)</sub>**.

> If the `nextTimestamp` is 10, it means that 9 is the latest timestamp that is given to some transaction **txn<sub>(some)</sub>** as its `commitTimestamp`.
> Does this mean that the commits with timestamp 9 have been applied to the storage?
>
> No, committing a transaction and actually applying its changes to the storage are two distinct events.
> Committing essentially marks a transaction as ready to be applied, but it doesn't instantly reflect in the storage. Committed transactions with assigned commitTimestamps form a queue,
> awaiting their turn to be applied to the storage in a serial manner. This means transactions are processed and reflected in the storage one after another, ensuring orderly updates.
>
> Implementations like [BadgerDb](https://github.com/dgraph-io/badger/blob/6acc8e801739f6702b8d95f462b8d450b9a0455b/txn.go#L104) wait
> till all the transactions upto the assigned `beginTimestamp` are applied.

We will look at the `WaitForMark(...)` later, but it waits for all the commits till `beginTimetamp` to be applied to the state machine.

> Is waiting really necessary in getting `beginTimestamp`?
> There are two options:
> 
> 1. Wait for a fixed timeout and allow all the commits till `beginTimestamp` to be applied to the state machine. This may increase the response time
> for read operations.
> 
> 2. Do not wait for the commits till `beginTimestamp` to be applied to the state machine. This may result result in an old value for a key. Example,
> if the `nextTimestamp` is 10, the current transaction will be awarded 9 as its `beginTimestamp`. However, there might be transactions in the queue
> waiting to be applied to state machine. This means, reading the value for a key from the state machine will get the value with latest version (/timestamp)
> which may be less than 9. 

The `Get` method is simple:

```go
func (transaction *ReadonlyTransaction) Get(key []byte) (mvcc.Value, bool) {
	versionedKey := mvcc.NewVersionedKey(key, transaction.beginTimestamp)
	return transaction.memtable.Get(versionedKey)
}
```

It involves the following:
1. Creating a new instance of `VersionedKey` that consists of the actual key and the `beginTimestamp` of the transaction.
2. Reading the value from the `MemTable`. This implementation uses `SkipList` as the in-memory data structure for storing versioned keys, so the `Get`
follows the algorithm described [earlier](#skiplist-and-mvcc).

This implementation shows reading the value for a key from an in-memory SkipList. The concept would remain the same even for implementing a `Get` 
in a persistent [LSM-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) based Key/Value storage engine. 
Each `ReadonlyTransaction` will get a `beginTimestamp` and instead of just reading from an in-memory MemTable, we will read 
from active memtable, inactive memtable(s) and the increasing levels of SSTable.

#### Implementing ReadWriteTransaction

#### Implementing WaterMarks

### References

- [BadgerDB](https://github.com/dgraph-io/badger)
- [SkipList](https://kt.academy/article/pmem-design-choices-and-use-cases#selective-persistence)
- [A critique of snapshot isolation](https://dl.acm.org/doi/10.1145/2168836.2168853)