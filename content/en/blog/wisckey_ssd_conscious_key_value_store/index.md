---
author: "Sarthak Makhija"
title: "WiscKey: Separating Keys from Values in SSD-Conscious Storage"
date: 2023-03-10
description: "LSM-tree (Log structured merge tree) is a data structure typically used when dealing with write-heavy workloads. LSM-tree optimizes the write-path by performing sequential writes to disk. WiscKey is a persistent LSM-tree-based key-value store that separates keys from values to minimize read and write amplification. The design of WiscKey is highly SSD optimized, leveraging both the sequential and random performance characteristics of the device."
tags: ["Storage engine", "LSM-tree", "WiscKey", "SSD-conscious"]
thumbnail: /wisckey.jpg
caption: "Background by Alex Conchillos on Pexels"
---

> LSM-tree (Log structured merge tree) is a data structure typically used for write-heavy workloads. LSM-tree optimizes the write path by performing sequential writes to disk. WiscKey is a persistent LSM-tree-based key-value store that separates keys from values to minimize read and write amplification. The design of WiscKey is highly SSD optimized, leveraging both the device's sequential and random performance characteristics.

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

Every `put(key, value)` goes in a log file and the active in-memory memtable. Once the active memtable is full, LevelDB *switches to a new memtable and log file* to handle further writes.

The previously active memtable is stored as an immutable memtable in RAM and flushed to disk in the background. This flush to disk generates a new SSTable file (about 2 MB) at level-0 and also discards the previous log file.

The size of all files in each level is limited, and increases by a factor of ten with the level number. For example, the size limit of all files at L1 is 10 MB, while the limit of L2 is 100 MB.

In order to perform the `get(key)` operation, LevelDB performs the following steps:

1. Perform `get` in the active memtable (RAM)
2. If not found, perform `get` in the immutable memtable (RAM)
3. If not found, perform `get` in the files from Level0 to Level6. LevelDB ensures that the keys do not overlap in the files from Level1 to Level6 whereas keys in the Level0 files can overlap.
   1. This means that a `get` operation may involve multiple files in Level0 and one file at each level from Level1 to Level6

> LevelDB implements the in-memory components using Skip list.
   
The below image represents the high-level architecture of LevelDB.

<img class="align-center" src="/leveldb.png" />

> [BadgerDB](https://github.com/dgraph-io/badger) maintains an array of immutable memtables. The `get` operation searches the active memtable and if the key is not found, it searches the inactive (or the immutable) memtables in the reverse order (the newest immutable memtable to the oldest immutable memtable).   

Let's spend a couple of minutes to understand the structure of the data block of an SSTable file before we move on.

SSTables contain key-value pairs sorted by key. Key-value pairs are encoded before they can be written to a file. One encoding scheme could be to use `key-size`, followed by `value-size`, followed by the `actual key` and then the `actual value` as depicted in the following table:

| **Key size (u32)** 	 | **Value size (u32)** 	 | **Key ([]byte)** 	 | **Value ([]byte)** 	 |
|----------------------|------------------------|--------------------|----------------------|

*We use `u32` type as an example for the key and the value size. Key-size and value-size would be represented using fixed-size data types like `u16` or `u32`*.

A naive way to get the value for a key in an SSTable would be to read the entire SSTable file, match each key and return the value if the matching key is found.

> **How do you read a single key-value pair from an SSTable file that follows the above-mentioned encoding scheme**? To read one key-value pair from an SSTable file, we need to read the first four bytes (`u32`) to get the key size, next four bytes to get the value size, then read the number of bytes equal to the key size to get the key and finally read the number of bytes equal to the value size to get the value.

SSTable files are organized into blocks (or sections). Typically, SSTable files in all the storage engines `index-block` + `bloom-filter-block` + `data-block`. *Imagine all these blocks as byte arrays ranging from one offset to the other.*

**Index-block** contains the key-offset pairs in the sorted order by key. The offset here is the offset of the key-value pair in the data block of the SSTable file. 

**Bloom filter block** contains the bloom filter byte array corresponding to all the keys present in the SSTable file

**Data block** contains the key-value pairs in the sorted order by key

> A Bloom filter is a probabilistic data structure used to test whether an element is a set member. A bloom filter can query against large amounts of data and return either “possibly in the set” or “definitely not in the set”. More information on bloom filter is available [here](/blog/bloom_filter/).

The previous image represents that a `get` operation may have to read multiple files. What we need is a way to establish the relationship between the amount of data read (or written to) and the amount of data requested by the user.
This relationship can later be used to question if the storage engine is doing too much IO or is the throughput suffering because of the additional IO? 

Let's understand "read and write amplification" to build such a relationship.

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

### Analysis of Read-Write amplification in LevelDB

Let's analyze the read and write amplification in LevelDB. 

**Let's start with read amplification**.

*LevelDB ensures that the keys do not overlap in the files from Level1 to Level6 whereas keys in the Level0 files can overlap.* 

Let's assume that the value for a key is not found in the active and the immutable memtable. Now, LevelDB needs to read SSTable files. In the worst case, LevelDB needs to check eight files in Level0, and one file for each of the remaining six levels: **a total of 14 files**. 

To find the value for a key within an SSTable file, LevelDB needs to read multiple metadata (or index) sections within the file. Specifically, the amount of data actually read is: 16-KB index section, a 4-KB bloom-filter section, and a 4-KB data section `(24KB)`.

> The read amplification in LevelDB is 336 (`14 files * 24KB`). Another way to state this is: "LevelDB amplifies the reads by 336 times in the worst case".

**Why is the size of the index-section 16KB?** 

SSTables can contain more than one metadata or index sections. Even if the index section is just 1KB, Block IO will involve reading the entire disk block from the underlying storage. LevelDB has multiple index sections, reading each section involves reading a 4KB disk block. Hence, the data that is actually read for the index section is 16KB. 

**Why is the size of the data-section just 4KB?** 

The data section can be huge but LevelDB needs to read just the 4KB data section to find the value for a key. The steps include: 

- LevelDB will load the index section to identify the position of the bloom-filter section.
  - Index section (or the index block) contains metadata and one of the metadata items could be the beginning offset of the bloom-filter block, other could be the offset of the keys in the data-block as mentioned earlier.
- LevelDB will now load the bloom-filter section and skip the file if the key is definitely not present.
- If the bloom filter indicates that the key may be present, LevelDB will identify the offset of the key from the index section.
- LevelDB jumps to the identified offset within the data section. With Block IO, the entire block containing the offset needs to be read. So, LevelDB ends up reading 4KB of the data block.   

*I am using the term section with index, bloom-filter and data to avoid confusing it with the disk block*. 

**Let's analyze the write amplification**.

LSM-tree based storage engines need to perform file merges during compaction. <<<<Compaction link>>>>

In LevelDB, the size of the level L<sub>i</sub> is 10 times the size of the level L<sub>i-1</sub> (Size of Level2 is 100MB whereas the size of Level1 is 10MB). To merge a file from the level L<sub>i-1</sub> to L<sub>i</sub>, LevelDB may end up reading 10 files from 
the level L<sub>i</sub> in the worst case and write back those 10 files after sorting. So, the write amplification of moving a file from one level to the next can be as high as 10. 

> In the worst case, a file may move from Level0 to Level6 through a series of compaction steps. The total write amplification in the worst case can be over 50 (write amplification is 10 for each level between L1 to L6). 

*We ignore the cost of moving a file from Level0 to Level1*.

**Why do we need an index-block in an SSTable file**?

In a key-value storage engine, the value is considered to be much bigger than the key. 

Index block is going to contain the key-offset pairs sorted by keys. Let's assume 10,000 keys in an SSTable file where each key is 100 bytes. This means each key-offset pair will take `104 bytes` (`100 bytes for the kyy + 4 bytes for the offset`, ignoring the encoding). 
So, the total size of the index block would be around 0.99 MB `(((104 * 10,000)/1024)/1024)`. To get the value for a key, the storage engine will need to read ~1 MB of index block and if the key is found, a random seek to the identified offset within the data block will be needed.

If the index-block were not there, the same get operation would have happened against the data block. Assuming the size of a single value to be 1024 bytes, we will have around 10.71 MB `((((100 + 1024) * 10,000)/1024)/1024)` as the size of the data block.

Before moving on, let's quickly recap the amplification numbers. The read amplification in LevelDB is 336 and the write amplification can be over 50, in the worst case.

> A key idea while designing a storage engine is to minimize the read and write amplification. Typically, SSDs can wear out through repeated writes, the high write amplification in LSM-trees can significantly reduce device lifetime.

### SSD considerations

There are fundamental differences between SSDs and HDDs which should be considered when designing a storage engine. Let's look at some of the most important considerations:
1. SSDs can wear out through repeated writes, the high write amplification in LSM-trees can significantly reduce the device lifetime. 
2. SSDs offer a large degree of internal parallelism

Point 1 means we should try to **reduce the write amplification** when designing a storage engine for SSD-conscious storage.

Point 2 means we should try to **leverage the parallelism offered by SSDs** when performing IO operations. Let's look at the below graph. For the request size >= 64KB, the aggregate throughput of random reads with 32 threads matches the sequential read throughput. 

<figure>
    <img class="align-center" src="/SSD-parallelism.png" />
</figure>

**Sequential and Random Reads on SSD**. This figure shows the sequential and random read performance for various request sizes on a modern SSD device. All requests are issued to a 100-GB file on ext4.

### WiscKey proposal

### Challenges with WiscKey

### Implementation of WiscKey

### References

- [WiscKey](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)
- [LSM tree](https://segmentfault.com/a/1190000041198407/en)
- [Introducing persistent memory](https://kt.academy/article/pmem-introducing-persistent-memory)
