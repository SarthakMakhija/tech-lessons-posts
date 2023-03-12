---
author: "Sarthak Makhija"
title: "WiscKey: Separating Keys from Values in SSD-Conscious Storage"
date: 2023-03-10
description: "LSM-tree (Log structured merge tree) is a data structure typically used when dealing with write-heavy workloads. LSM-tree optimizes the write-path by performing sequential writes to disk. WiscKey is a persistent LSM-tree-based key-value store that separates keys from values to minimize read and write amplification. The design of WiscKey is highly SSD optimized, leveraging both the sequential and random performance characteristics of the device."
tags: ["Storage engine", "LSM-tree", "WiscKey", "SSD-conscious"]
thumbnail: /wisckey.jpg
caption: "Background by Alex Conchillos on Pexels"
---

> [LSM-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree#:~:text=In%20computer%20science%2C%20the%20log,%2C%20maintain%20key%2Dvalue%20pairs.) (Log structured merge tree) is a data structure typically used for write-heavy workloads. LSM-tree optimizes the write path by performing sequential writes to disk. WiscKey is a persistent LSM-tree-based key-value store that separates keys from values to minimize read and write amplification. The design of WiscKey is highly SSD optimized, leveraging both the device's sequential and random performance characteristics.

This article summarises the [WiscKey](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf) paper published in 2016.

Before we understand the paper, it is essential to understand the LSM-tree data structure, read and write amplification in the LSM-tree and various SSD characteristics that should be considered while building SSD-conscious storage engine.

### LSM-tree

LSM-tree is a write-optimized data structure implemented by storage engines for supporting write-heavy workloads. A lot of storage engines including [BadgerDB](https://github.com/dgraph-io/badger), [RocksDB](https://github.com/facebook/rocksdb) and
[LevelDB](https://github.com/google/leveldb) use LSM-tree as the core data structure.

> Storage engine is a software module that provides data structures for efficient reads and writes. The two most common data structures are B+Tree (read-optimized) and LSM-tree (write-optimized).

Let's look at the structure of LSM-tree to understand why it is write-optimized. LSM-tree consists of components of exponentially increasing sizes, C0 to Ck. C0 is a *RAM resident* component that stores key-value pairs sorted by key and supports efficient writes and reads, whereas C1 to Ck are *immutable disk-resident* components sorted by key.

<img class="align-center" src="/lsm-c0-ck.png" />

> **Data structure choice for C0**: The data structure for C0 should maintain the keys in the sorted order and provide efficient reads and writes. This data structure could be a [treemap](https://docs.oracle.com/javase/8/docs/api/java/util/TreeMap.html) or a [red-black tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree) or a [skip list](https://en.wikipedia.org/wiki/Skip_list). Of all these data structures, the Skip list is the one that supports versioned keys, same key with different versions.

LSM-tree buffers the data in-memory and performs a sequential write to disk after the in-memory buffer is full. The below image highlights the throughput difference between sequential and random writes on an NVMe SSD, the difference would be a lot higher on an HDD. 

<figure>
    <img class="align-center" src="/sequential-random-write.png" /> 
    <figcaption class="figcaption">Sequential write throughput > Random write throughput</figcaption>
</figure>

> LSM-tree based storage engines offer better write throughput because they perform sequential writes to disk.

Let's take a look at the overall flow of `put(key: []byte, value: []byte)` and `get(key: []byte)` operations in LSM-tree.

Every `put(key, value)` in the LSM-tree adds the key-value pair in the C0 component and after C0 is full, the entire data is flushed to disk. LSM-trees treat `delete(key)` as another `put` which will put the key with a deleted marker. 

After C0 is full, the entire data is flushed to disk. Simply speaking, the entire in-memory data consisting of key-value pairs is encoded in a byte array and written to disk. If the C1 component already exists on disk,   
the buffered content is merged with the contents of C1. **More on this later. <<<<<<<<<<<<<<<<<<<>>>>>>>>>>>>>**

To ensure persistent writes, every `put(key, value)` *appends* the key-value pair to a [WAL](https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html) file and then writes the key-value pair to the C0 component. Appending to a WAL file is also a sequential write to disk.   

Every `get(key)` in the LSM-tree goes through the RAM based component to disk components from C1 to Ck in the order. The `get(key)` operation first queries the C0 component, if the value for the key is not found, the search proceeds
to the disk resident component C1. This process continues until the value is found or all the disk resident components have been scanned. LSM-trees may need multiple reads for a point lookup. Hence, LSM-trees are most useful when inserts are more common than lookups. 

**Quick summary**
> LSM-tree is a collection of exponentially increasing sized components, C0 to Ck. <br/> Writes go to C0 (in RAM). <br/> After C0 is full, the entire data is flushed to disk. <br/> The get operation involves going through C0 to Ck.

Let's look at the structure of LSM-tree in LevelDB to understand a bit more on C0 to Ck components. 

### LevelDB

LevelDB is a key-value storage engine based on LSM-tree. LevelDB maintains the following data structures:
1. On-disk log file (WAL) to persist the writes
2. Two in-memory components called "memtables" (active and passive)
3. Seven levels of on-disk components called "SSTable" (Sorted string table) 

Every `put(key, value)` goes in a log file and the active in-memory memtable. Once the active memtable is full, LevelDB *switches to a new memtable and log file* to handle further writes.

The previously active memtable is stored as an immutable memtable in RAM and flushed to disk in the background. This flush to disk generates a new SSTable file (about 2 MB) at level-0 and also discards the previous log file (WAL). 

The size of all files in each level is limited, and increases by a factor of ten with the level number. For example, the size limit of all files at L1 is 10 MB, while the limit of L2 is 100 MB.

In order to perform the `get(key)` operation, LevelDB performs the following steps:

1. Perform `get` in the active memtable (RAM),
2. If not found, perform `get` in the immutable memtable (RAM),
3. If not found, perform `get` in the files from Level0 to Level6. LevelDB ensures that the keys do not overlap in the files from Level1 to Level6 whereas keys in the Level0 files can overlap.
   1. This means that a `get` operation may involve multiple files in Level0 and one file at each level from Level1 to Level6

> LevelDB implements the in-memory components using [Skip list](https://en.wikipedia.org/wiki/Skip_list).
   
The below image represents the high-level architecture of LevelDB.

<img class="align-center" src="/leveldb.png" />

> [BadgerDB](https://github.com/dgraph-io/badger) maintains an array of immutable memtables instead of one immutable memtable to optimize reads. The `get` operation searches the active memtable and if the key is not found, it searches the inactive (or immutable) memtables in the reverse order (the newest immutable memtable to the oldest immutable memtable).   

Let's spend a couple of minutes to understand the structure of an SSTable file before we move on.

SSTables contain key-value pairs sorted by key. Key-value pairs are encoded before they can be written to a file. One encoding scheme could be to use `key-size`, followed by `value-size`, followed by the `actual key` and then the `actual value` as depicted in the following table:

| **Key size (u32)** 	 | **Value size (u32)** 	 | **Key ([]byte)** 	 | **Value ([]byte)** 	 |
|----------------------|------------------------|--------------------|----------------------|

*We use `u32` type as an example for the key and the value size. Key-size and value-size would be represented using fixed-size data types like `u16` or `u32`*.

A naive way to get the value for a key in an SSTable would be to read the entire SSTable file, match each key and return the value if the matching key is found.

> **How do you read a single key-value pair from an SSTable file that follows the above-mentioned encoding scheme**? To read one key-value pair from an SSTable file, we need to read the first four bytes (`u32`) to get the key size, next four bytes to get the value size, then read the number of bytes equal to the key size to get the key and finally read the number of bytes equal to the value size to get the value.

SSTable files are organized into blocks (or sections). Typically, SSTable files in all the LSM-tree based storage engines include `index-block` + `bloom-filter-block` + `data-block`. *Imagine all these blocks as byte arrays ranging from one offset to the other.*

**Index-block** contains the key-offset pairs in the sorted order by key. The offset here is the offset of the key-value pair in the data block of the SSTable file. 

**Bloom filter block** contains the bloom filter byte array of all the keys present in the SSTable file

**Data block** contains the actual data; the key-value pairs in the sorted order by key

> A Bloom filter is a probabilistic data structure used to test whether an element is a set member. A bloom filter can query against large amounts of data and return either “possibly in the set” or “definitely not in the set”. More information on bloom filter is available [here](/blog/bloom_filter/).

LSM-trees may end up reading a lot of files (or portions of files) in order to perform a `get` operation. What we need is a way to establish the relationship between the amount of data read (or written to) and the amount of data requested by the user.
This relationship can later be used to question if the storage engine is doing too much IO or is the throughput suffering because of the additional IO? 

**Quick summary**
> LevelDB uses WAL, memtables and SSTables as its data structures. <br/> Memtable is implemented using Skip list <br/> SSTables are sorted string tables and organized into levels, Level0 to Level6. <br/> SSTables are organized into index-block, bloom-filter block and data block. <br/> Puts go to WAL and the active memtable, gets go to active memtable -> immutable memtable -> SSTables. 

Let's now understand "read and write amplification".

<<<<Compaction>>>

### Read-Write amplification

HDDs, SDDs and NVMe SSDs are block storage devices. The smallest unit of exchange between an application and a block storage device is a block. Usually the size of a block is 4KB.
Let's assume a key/value storage engine on a block storage device like HDD. When the key/value storage engine wants to fetch the value for a 128 bytes key, it must do the following:
- read the entire block containing the value from the underlying device (*ignore how that block is located*),
- allocate a buffer in-memory to hold the result

Even when making a small update say, 128 bytes, the storage engine needs to do the following:
- read the entire block containing those 128 bytes into a memory buffer,
- update 128 bytes in the buffer,
- write the entire block to the underlying device.

> Block IO involves transferring an entire block of data, typically 4K bytes at a time. So the task to read 128 bytes would require reading at least one block of 4K bytes. 

This results in "amplification". Both the reads and the writes are amplified. The storage engine has to read (and write to) more than what is requested by the user.

**Read amplification** is the ratio of the amount of data read from the storage device and the amount of data requested by the user. In the above example, read amplification is 32 (`4*1024 bytes / 128 bytes`). 

**Write amplification** is the ratio of the amount of data written to the storage device and the amount of data requested by the user. In the above example, write amplification is 32 (`4*1024 bytes / 128 bytes`).

**Quick summary**
> The smallest unit of data exchange between an application and a block storage device is a block, usually 4KB in size. <br/> Read/Write amplification is the amount of data read from (or written to) the storage device and the amount of data requested by the user.

### Analysis of Read Write amplification in LevelDB

Let's analyze the read and write amplification in LevelDB. 

**Let's start with read amplification**.

*LevelDB ensures that the keys do not overlap in the files from Level1 to Level6 whereas keys in the Level0 files can overlap.* 

Let's assume that the value for a key is not found in the active and the immutable memtable. Now, LevelDB needs to read SSTable files. In the worst case, LevelDB needs to check eight files in Level0, and one file for each of the remaining six levels: **a total of 14 files**. 

To find the value for a key within an SSTable file, LevelDB needs to read multiple metadata (or index) sections within the file. Specifically, the amount of data actually read is: 16-KB index section, a 4-KB bloom-filter section, and a 4-KB data section `(24KB)`.

> The read amplification in LevelDB is 336 (`14 files * 24KB`). Another way to state this is: "LevelDB amplifies the reads by 336 times in the worst case".

**Let's analyze the write amplification**.

LSM-tree based storage engines need to perform file merges during compaction. <<<<Compaction link>>>>

In LevelDB, the size of the level L<sub>i</sub> is 10 times the size of the level L<sub>i-1</sub> (Size of Level2 is 100MB whereas the size of Level1 is 10MB). To merge a file from the level L<sub>i-1</sub> to L<sub>i</sub>, LevelDB may end up reading 10 files from 
the level L<sub>i</sub> in the worst case and write back those 10 files after sorting. So, the write amplification of moving a file from one level to the next can be as high as 10. 

> In the worst case, a file may move from Level0 to Level6 through a series of compaction steps. The total write amplification in the worst case can be over 50 (write amplification is 10 for each level between L1 to L6). 

*We ignore the cost of moving a file from Level0 to Level1*.

#### Why is the size of the index-section 16KB?

SSTables can contain more than one metadata or index sections. Even if the index section is just 1KB, Block IO will involve reading the entire disk block from the underlying storage. LevelDB has multiple index sections and reading each section involves reading a 4KB disk block. Hence, the data that is actually read for the index section is 16KB.

#### Why is the size of the data-section just 4KB?

The data section can be huge but LevelDB needs to read just the 4KB data section to find the value for a key. The steps include:

- LevelDB will load the index section to identify the position of the bloom-filter section.
    - Index section (or the index block) contains metadata and one of the metadata items could be the beginning offset of the bloom-filter block, (other could be the offset of the keys in the data-block as mentioned earlier).
- LevelDB will now load the bloom-filter section and skip the file if the key is definitely not present.
- If the bloom filter indicates that the key may be present, LevelDB will identify the offset of the key from the index section.
- LevelDB jumps to the identified offset within the data section. With Block IO, the entire block containing the offset needs to be read. So, LevelDB ends up reading 4KB of the data block.

*I am using the term section with index, bloom-filter and data to avoid confusing it with the disk block*.

#### Why do we need an index-block in an SSTable file?

In a key-value storage engine, the value is considered to be much bigger than the key. 

Index block is going to contain the key/value-offset pairs sorted by keys. Let's assume 10,000 keys in an SSTable file where each key is 100 bytes long. This means each key-offset pair will take `104 bytes` (`100 bytes for the key + 4 bytes for the offset`, ignoring the encoding). 
So, the total size of the index block would be around 0.99 MB `(((104 * 10,000)/1024)/1024)`. To get the value for a key, the storage engine will need to read ~1 MB of index block and if the key is found, a random seek to the identified offset within the data block will be needed.

If the index-block were not there, the same get operation would have happened against the data block. Assuming the size of a single value to be 1024 bytes, we will have around 10.71 MB `((((100 + 1024) * 10,000)/1024)/1024)` as the size of the data block.

**Quick summary**
> The **read amplification in LevelDB is 336** and the **write amplification can be over 50**, in the worst case. <br/> A key idea while designing a storage engine is to minimize the read and write amplification to reduce IO latency. Also, SSDs can wear out through repeated writes, the high write amplification in LSM-trees can significantly reduce device lifetime.

### SSD considerations

There are fundamental differences between SSDs and HDDs which should be considered when designing a storage engine. Let's look at some of the most important considerations:
1. SSDs can wear out through repeated writes, the high write amplification in LSM-trees can significantly reduce the device lifetime. 
2. SSDs offer a large degree of internal parallelism

Point 1 means we should try to **reduce the write amplification** when designing a storage engine for SSD-conscious storage. We have already seen that [compaction](#analysis-of-read-write-amplification-in-leveldb) process is the source of high write-application in LSM-tree based storage engines.

Point 2 means we should try to **leverage the parallelism offered by SSDs** when performing IO operations. Let's look at the below graph. For the request size >= 64KB, the aggregate throughput of random reads with 32 threads matches the sequential read throughput. 

<figure>
    <img class="align-center" src="/SSD-parallelism.png" />
</figure>

**Sequential and Random Reads on SSD**. This figure shows the sequential and random read performance for various request sizes on a modern SSD device. All requests are issued to a 100-GB file on ext4.

**Quick summary**
> When designing a storage engine for SSDs, try to reduce the write-amplification and leverage the internal parallelism offered by SSDs.

With these considerations, let's understand WiscKey proposal.

### WiscKey proposal

WiscKey proposes four key ideas:
1. Separate values from keys, keeping only the keys in the LSM-tree, putting values in a separate value-log.
2. Leverage the parallel random read characteristic of SSDs during range queries.
3. Introduce garbage collection to remove values corresponding to deleted keys from value-log.
4. Remove LSM-tree log (WAL log) without sacrificing consistency (under [Optimizations](#optimizations)).

Let's discuss each of these ideas one by one.

#### Separate values from keys

Compaction is the reason for the major performance cost in LSM-trees. During compaction, multiple SSTable files are read in-memory, sorted and written back. This is also the reason
for the high write amplification in LevelDB. If we look at the compaction process carefully, we will realize that this process only needs to sort the keys, while values can be managed separately.
Since keys are usually smaller than values, compacting only keys could significantly reduce the amount of data needed during the sorting.

In WiscKey, only the location of the value is stored in the LSM-tree along with the key, while the actual values are stored in a separate value-log file.

The `put(key, value)` operation in WiscKey makes a small modification to the original `put` flow. Every `put(key, value)` in the WiscKey adds the `key-value pair` in the `value-log` and then adds the `key` along with the value-log offset in the memtable.
Converting the active memtable to the immutable memtable, flushing the active memtable to disk and performing compaction process in the background remains the same. 

> Memtables and SSTables in WiscKey contain keys along with the key-value pair offset from value-log. Given, keys are smaller than values, the amount of data needed during compaction is significantly reduced.

Let's look at the flow of the `get(key)` operation. 

1. Perform `get` in the active memtable (RAM),
2. If not found, perform `get` in the immutable memtable (RAM),
3. If not found, perform `get` in the SSTable files.

If the `get` operation finds the key, a random seek to the key-value pair offset needs to be performed in the value-log to get the value. 
This requires an additional IO for every `get` operation. The research paper claims that the LSM-tree of Wisckey is a lot smaller than LevelDB, a lookup may search fewer levels of table files and a significant portion of the LSM-tree can be easily cached in memory.

> BadgerDB is an implementation of the WiscKey paper, but it makes a small modification to `put` and the `get` operations. If the value size is less than some threshold, the value will be put in the memtable else the value-offset will be put in the memtable. If the key is found during the `get` operation, BadgerDB loads the value from the value-log if the retrieved value is a value-offset. SSTables always contain the value-offset.   

Deleting a key in WiscKey will delete it from the LSM-tree but the value-log remains untouched. WiscKey proposes [garbage collection](#introduce-garbage-collection) to remove invalid (or dangling) values from value-log.

#### Leverage the internal parallelism of SSDs

All the modern storage engines provide support for range queries. LevelDB provides the clients with an `iterator-based` interface with `Seek(key)`, `Next()`, `Prev()`, `Key()` and `Value()` operations. To scan a range of key-value pairs, the client can first `Seek(key)` to the starting key, then call `Next()` or `Prev()` to search keys one by one. To retrieve the key or the value of the current iterator position, the client calls `Key()` or `Value()`, respectively.

In LevelDB, since keys and values are stored together and sorted, a range query can sequentially read key-value pairs from SSTable files. However, since keys and values are stored separately in WiscKey, range queries require random reads (from value-log), and are hence not efficient.

Wisckey leverages the same iterator-based interface for range queries as LevelDB but to make range queries efficient, WiscKey leverages the parallel I/O characteristic of SSD devices to prefetch values from the value-log during range queries. 

WiscKey tracks the access pattern of a range query. Once a contiguous sequence of key-value pairs is requested, WiscKey starts reading the following keys from the LSM-tree sequentially. The values corresponding to those prefetched keys are resolved in parallel (using multiple threads) from the value-log.

> BadgerDB uses `PrefetchSize (int)` and `PrefetchValues (bool)` in the `IteratorOptions` struct [`iterator.go`] to decide if the values have to be prefetched. The `DefaultIteratorOptions` defines 100 as the prefetch size.

#### Introduce garbage collection 

Wisckey stores only the keys in the LSM-tree and the values are managed separately. During compaction, the deleted keys will be removed from SSTable files but the values corresponding to the deleted keys will still be present in the value-log. Let's call those values as dangling values.
To deal with dangling values, WiscKey proposes a lightweight garbage collector to reclaim free space in the value-log.

Let's look at the structure of the value-log. Every entry in the value-log contains `key-size`, `value-size`, `key` and the `value`. The `head` end of this log corresponds to the end of the value-log where new values will be appended and the `tail` of this log is where garbage collection starts freeing space whenever it is triggered. 
Only the part of the value-log between the tail and the head offset contains valid values and will be searched during lookups.

<img class="align-center" src="/value-log.png" />

Garbage involves the following steps:
1. WiscKey reads a chunk of key-value pairs (several MBs) from the `tail` offset of the value-log. It then finds which of those values are valid by querying the LSM-tree. 
2. After all the valid key-value pairs have been identified, the entire byte array containing the valid key-value pairs is appended to the end of the log.
3. To avoid losing any data in case a crash happens during garbage collection, WiscKey calls `fsync` on the value-log.
4. WiscKey needs to add the updated value’s addresses to the LSM-tree. So, it adds the key & value-offset pairs to the LSM-tree along with the current tail offset. The tail is stored in the LSM-tree as `<'tail', tail-value-log-offset>`.
5. Finally, the free space is reclaimed and the head offset is stored in the LSM-tree.

**Quick summary**
> WiscKey separates values from keys in the LSM-tree. LSM-tree contains keys and the offsets of the key-value pair from the value-log. This reduces the write-amplification during compaction. </br> WiscKey leverages the internal parallelism offered by SSDs, during range queries. <br/> Wisckey introduces garbage collection to remove dangling values from the value-log.

Let's discuss various optimizations proposed in the WiscKey paper.

### Optimizations

WiscKey offers two optimizations:
1. **Value-Log Write Buffer**: to buffer the key-value pairs before they are written to the value-log
2. **Remove LSM-tree log**: use value-log as recovery mechanism

#### Value-Log Write Buffer

For each `put(key, value)` operation, WiscKey needs to append the key-value pair in the value-log. This means every `put` operation will require a `write` system call. With large writes (larger than 4 KB), the device throughput is fully utilized.

To reduce the write-overhead, WiscKey buffers the incoming key-value pairs in-memory and the buffer is flushed to value-log only when the buffer size exceeds a threshold or when the user requests a synchronous insertion.
This requires a change in the `get` operation. 

Assume that a `get` operation is able to find the value offset in the active memtable. Now, the system needs to look up the value-offset in the value-log to get the value. With the introduction of the value-log buffer, 
the lookup operation will be performed in the value-log buffer first and if the value-log offset is not found in the buffer, it actually reads from the value-log.

*This optimization means that the buffered data can be lost during a crash.* 

#### Remove the LSM-tree Log

The value-log is storing the entire key-value pair. If a crash happens before the keys are persistent in the LSM-tree (and after they have been written to the value-log), they can be recovered by scanning the value-log. 
In order to optimize the recovery of the LSM-tree from the value-log, the `head` offset of the value-log is periodically stored in the LSM-tree as `<'head', head-value-log-offset>`.
When the database is opened, WiscKey starts the value-log scan from the most recent head position stored in the LSM-tree, and continues scanning until the end of the value-log.
So, removing the LSM-tree log is a safe optimization.

### Reference implementation of WiscKey

[BadgerDB](https://github.com/dgraph-io/badger) is an implementation of the WiscKey paper. We will briefly look at the `put` and the `get` implementations of BadgerDB.

Let's start with `put(key, value)`.

```golang
//Fields ommitted
type request struct {
	// Input values
	Entries []*Entry

	Ptrs []valuePointer
}
type Entry struct {
	Key       []byte
	Value     []byte
}
type valuePointer struct {
	Fid    uint32
	Len    uint32
	Offset uint32
}

//Code ommitted
func (db *DB) writeRequests(requests []*request) error {
	db.opt.Debugf("writeRequests called. Writing to value log")
	
	//write the requests to the value log file
	err := db.vlog.write(requests) 
	if err != nil {
		done(err)
		return err
	}
	for _, request := range requests {
		//write a single request to the LSM-tree
		if err := db.writeToLSM(request); err != nil {  
			done(err)
			return y.Wrap(err, "writeRequests")
		}
	}
	return nil
}
```

BadgerDB implements snapshot transaction isolation. Let's assume the `commit()` method on the transaction is invoked. The commit operation results in calling the `writeRequests`
method on `db` through a single goroutine in a fashion that is very much similar to a [singular update queue](https://martinfowler.com/articles/patterns-of-distributed-systems/singular-update-queue.html).

As a part of this method, the key-value pairs are written to the value-log and then each request is written to the LSM-tree. 

Let's look at the `writeToLSM` method to understand the content of the memtable.

```golang
//Code ommitted
func (db *DB) writeToLSM(reqest *request) error {
    //Iterate through all the entries in the request
	for i, entry := range reqest.Entries {
		var err error
		if db.opt.managedTxns || entry.skipVlogAndSetThreshold(db.valueThreshold()) {
			//Write the entire key and the value in the memtable. 
			//Memtable is implemented using Skip list 
			err = db.memtable.Put(entry.Key,
				y.ValueStruct{
					Value: entry.Value,
					Meta:      entry.meta &^ bitValuePointer,
					UserMeta:  entry.UserMeta,
					ExpiresAt: entry.ExpiresAt,
				})
		} else {
			// Write the pointer to the memtable.
			err = db.memtable.Put(entry.Key,
				y.ValueStruct{
					Value:     reqest.Ptrs[i].Encode(),
					Meta:      entry.meta | bitValuePointer,
					UserMeta:  entry.UserMeta,
					ExpiresAt: entry.ExpiresAt,
				})
		}
		if err != nil {
			return y.Wrapf(err, "while writing to memTable")
		}
	}
	if db.opt.SyncWrites {
	    //Memtable contains both a WAL and a skip list. 
	    //Sync the writes to the WAL associated with the current memtable. 
		return db.memtable.SyncWAL()
	}
	return nil
}

func (e *Entry) skipVlogAndSetThreshold(threshold int64) bool {
	if e.valThreshold == 0 {
		e.valThreshold = threshold
	}
	return int64(len(e.Value)) < e.valThreshold
}
```

The method `writeToLSM` is writing the entire key-value pair in the memtable if the size of the value is less than some threshold, else the encoded value of the `value pointer` is written to the memtable. `ValuePointer` 
references the key-value pair offset in a value-log.

Let's look at the `get(key)` method.

```golang
//Code ommitted
type ValueStruct struct {
	Meta      byte
	UserMeta  byte
	ExpiresAt uint64
	Value     []byte
	Version uint64
}

func (txn *Txn) Get(key []byte) (item *Item, rerr error) {    
	item = new(Item)
	seek := y.KeyWithTs(key, txn.readTs)
	
	//Get the valueStruct corresponding to the key.
	//db.get will perform a get operation in all the memtables (active and all the immutable memtables), 
	//if the key is not found, it will perform a get across all the levels. 
	valueStruct, err := txn.db.get(seek)
	
	if valueStruct.Value == nil && valueStruct.Meta == 0 {
		return nil, ErrKeyNotFound
	}
	if isDeletedOrExpired(valueStruct.Meta, valueStruct.ExpiresAt) {
		return nil, ErrKeyNotFound
	}

    //if the key exists, return an item. 
    //Item will abstract the idea of fetching the value from the value-log   
	item.key = key
	item.version = valueStruct.Version
	item.meta = valueStruct.Meta
	item.userMeta = valueStruct.UserMeta
	item.vptr = y.SafeCopy(item.vptr, valueStruct.Value)
	item.txn = txn
	item.expiresAt = valueStruct.ExpiresAt
	return item, nil
}
```

The `Get(key)` method in the transaction does two things:
1. Invokes the `get` method of the `db` abstraction to get an instance of `ValueStruct`
2. If the key exists, it returns an `Item`. `ValueStruct` may or may not contain the value, so `Item` abstraction ensures that the value is fetched from the value-log if needed.

Let's look at the method `yieldItemValue` in the `Item`. The idea is to decode the value pointer (get the instance of `valuePointer` back from the byte array) and perform a random read
operation in the value-log.

```golang
//Code ommitted
func (item *Item) yieldItemValue() ([]byte, func(), error) {
	key := item.Key() // No need to copy.
	if !item.hasValue() {
		return nil, nil, nil
	}

	var vp valuePointer
	vp.Decode(item.vptr)
	
	db := item.txn.db
	//Read the value from the value log. This is a random seek in the value-log.
	result, cb, err := db.vlog.Read(vp, item.slice) 
	
	....
}
```

The article has already gone too long, but I would like to put a short mention of the `iterator` implementation in BadgerDB. 

BadgerDB provides a `NewIterator(opt IteratorOptions)` method that returns an iterator object. 

The challenge is: where should the iterator point to? Should it point to the active memtable? Or, which of the N immutable memtables or M SSTable files should it point to?

```golang
//Code ommitted
func (txn *Txn) NewIterator(opt IteratorOptions) *Iterator {
    tables, decr := txn.db.getMemTables()
    
    for i := 0; i < len(tables); i++ {
		iters = append(iters, tables[i].skiplist.NewUniIterator(opt.Reverse))
	}
	iters = append(iters, txn.db.levelController.iterators(&opt)...)
	res := &Iterator{
		txn:    txn,
		iitr:   table.NewMergeIterator(iters, opt.Reverse),
		opt:    opt,
		readTs: txn.readTs,
	}
}
```

BadgerDB creates iterators across all the objects: all the memtables and all the SSTable files from Level0 to the last level, and returns an instance of `MergeIterator`. The `MergeIterator` organizes all the 
iterators in the form of a binary tree.

```golang
//Code ommitted
type MergeIterator struct {
	left  node
	right node
}

//y.Iterator is an interface that has various implementations like skiplist.NewUniIterator, MergeIterator
func NewMergeIterator(iters []y.Iterator, reverse bool) y.Iterator {
	switch len(iters) {
	case 2:
		mi := &MergeIterator{
			reverse: reverse,
		}
		//When we are left with 2 iterators, set the left and the right node of MergeIterator
		mi.left.setIterator(iters[0])
		mi.right.setIterator(iters[1])
		mi.small = &mi.left
		return mi
	}
	mid := len(iters) / 2
	
	//Recursive call to arrange the iterators from 0 to mid-1, and then mid to the last index
	return NewMergeIterator(
		[]y.Iterator{
			NewMergeIterator(iters[:mid], reverse),
			NewMergeIterator(iters[mid:], reverse),
		}, reverse)
}
```

`Seek` by design is a recursive operation (imagine it to be a binary tree traversal). Each iterator may find some value for the key. (The same key may be present in the active memtable and in one SSTable.)
So, `mi.fix()` resolves the key that will be returned from the `MergeIterator`.

```golang
//Code ommitted
func (mi *MergeIterator) Seek(key []byte) {
	mi.left.seek(key)
	mi.right.seek(key)
	mi.fix()
}
```

MergerIterator is available [here](https://github.com/dgraph-io/badger/blob/main/table/merge_iterator.go).

### Conclusion

We have finally reached here :).

LSM-tree based storage engines typically include the following data structures:
1. On-disk log file (WAL) to persist the writes.
2. In-memory memtable(s).
3. On-disk files organized in levels.

LSM-trees offer higher write throughput because the writes are always sequential in nature, but reads are not so great because LSM-trees may have to scan multiple files or portions of multiple files.

When designing a storage engine for SSDs, we should consider SSD characteristics:
1. SSDs can wear out through repeated writes, the high write amplification in LSM-trees can significantly reduce the device lifetime.
2. SSDs offer a large degree of internal parallelism

The core ideas of WiscKey include separating values from keys in the LSM-tree to reduce write amplification and leveraging the parallel IO characteristic of SSD.

### References

- [WiscKey](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)
- [LSM-tree](https://segmentfault.com/a/1190000041198407/en)
- [Introducing persistent memory](https://kt.academy/article/pmem-introducing-persistent-memory)
- [BadgerDB](https://github.com/dgraph-io/badger)


