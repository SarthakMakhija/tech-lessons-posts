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

### Understanding Skip list

### Implementing Serialized Snapshot Isolation in a Key/Value store

#### Understanding MVCC

#### Implementing Get

#### Implementing Put

### References