---
author: "Sarthak Makhija"
title: "Building blocks of LSM based key/value storage engines: WAL"
date: 2024-10-10
description: "asdasdasdsadas"
tags: ["LSM", "Building blocks of LSM", "WAL"]
thumbnail: /bloomfilter_title.webp
caption: ""
---

The Write-Ahead Log (WAL) is a fundamental component of LSM-based key-value storage engines, ensuring data durability and 
enabling recovery from system failures. The concept is straightforward: a WAL is an append-only log file on disk. 
In an LSM-based storage engine, every write operation, whether it's a single key-value pair or a transactional batch, is first 
recorded in the WAL before being added to the active memtable. This append-only nature of WALs allows for efficient sequential disk 
access.

### Encode and append 

[Appending](https://github.com/SarthakMakhija/go-lsm/blob/main/log/wal.go#L71) to a WAL involves accepting a key-value pair or a 
transactional batch, encoding the data, and writing it to the file. The code for append looks like the following:

```go
var ReservedKeySize = int(unsafe.Sizeof(uint16(0)))
var ReservedValueSize = int(unsafe.Sizeof(uint16(0)))

func (wal *WAL) Append(key kv.Key, value kv.Value) error {
	buffer := make([]byte, key.EncodedSizeInBytes()+value.SizeInBytes()+ReservedKeySize+ReservedValueSize)

	binary.LittleEndian.PutUint16(buffer, uint16(key.EncodedSizeInBytes()))
	copy(buffer[block.ReservedKeySize:], key.EncodedBytes())

	binary.LittleEndian.PutUint16(buffer[block.ReservedKeySize+key.EncodedSizeInBytes():], uint16(value.SizeInBytes()))
	copy(buffer[block.ReservedKeySize+key.EncodedSizeInBytes()+block.ReservedValueSize:], value.Bytes())

	_, err := wal.file.Write(buffer)
	return err
}
```

The append operation in the WAL (Write-Ahead Log) involves handling variable-length data for 
[Key](https://github.com/SarthakMakhija/go-lsm/blob/737fa1ee1266bf9b96b454e729d63bbb9d7a928f/kv/key.go)/[Value](https://github.com/SarthakMakhija/go-lsm/blob/737fa1ee1266bf9b96b454e729d63bbb9d7a928f/kv/value.go) 
pairs. Since the file needs information about the key and value sizes for proper decoding during recovery, the code utilizes 
a specific encoding scheme.

1. **Size Precedence**:
- Two bytes are reserved at the beginning to store the size (encoded length) of the key.
- Another two bytes are reserved following the key to store the size of the value. 
This approach limits both key and value sizes to a maximum of 64KB (2 bytes can represent a maximum value of 65535).

2. **Encoding Format**:
The following format illustrates the structure of the data written to the WAL:

```go
 -------------------------------------------------------
| 2 bytes key size | kv.Key | 2 bytes value size | Value|
 -------------------------------------------------------
```

- The first two bytes represent the encoded size of the key.
- Following this is the actual encoded byte sequence of the key itself.
- Next two bytes contain the encoded size of the value.
- Finally, the remaining bytes represent the actual value data.

3. **Writing to the WAL File**:
Once the key-value pair is encoded according to the defined format, the entire byte sequence is written to the underlying WAL file.

Before moving further, I would like to refer to the post by [Alex Petrov](https://www.databass.dev/) on [Flavors of IO](https://medium.com/databasss/on-disk-io-part-1-flavours-of-io-8e1ace1de017#:~:text=There%20are%20several%20%E2%80%9Cflavors%E2%80%9D%20of,Vectored%20IO%3A%20writev%2C%20readv).

**Standard IO**: uses `read()` and `write()` system calls for performing IO operations. These system calls involve three components: 
Application, kernel page cache and the underlying block device. A file read from the application will check if the data is available
in the kernel page cache. If the data is present in the page cache, it is simply copied to the user-space, else the data 
is loaded from the underlying block device to the page cache and then returned to the user. A file write simply copies 
user-space buffer (/byte array) to kernel page cache, marking the written page as dirty. The data is not written to the disk immediately.

**Direct IO**: bypasses kernel page cache, thereby transferring the data from the application to the underlying device. While 
direct I/O can improve performance by reducing the number of context switches and memory copies, it doesn't guarantee that the 
data is immediately written to disk.

We need to ensure that the data modifications are transferred to the underlying device. Let's look at fsync. 

### Fsync

Durability in a storage system guarantees that data modifications persist even in the event of crashes or power failures. 
While a file write (`file.write(..)`) system call modifies OS page caches, it doesn't necessarily ensure the data is immediately 
transferred to the physical disk.

To achieve true durability, the operating system's page cache needs to be flushed to the underlying storage device. 
This is where the fsync system call comes into play. 
As documented in [man7.org](https://man7.org/linux/man-pages/man2/fsync.2.html), fsync forces the modified data to be written 
to the disk, ensuring its persistence even in system crashes or reboots.

While fsync guarantees data integrity, it comes at a cost. Each fsync call introduces a performance overhead as it waits for 
the write operation to complete on the disk. This can significantly impact application throughput, as the following figure 
demonstrates with performance figures from a 300GB NVMe SSD.

<figure>
    <img class="align-center" src="/fsync_and_without_fsync.png" /> 
    <figcaption class="figcaption">Throughput with fsync after each write and without fsync: m5d.2xlarge, 8 vCPU, 32 GB Memory, 300GB NVMe SSD </figcaption>
</figure>

A crucial question arises: when should the system perform fsync? Here's a breakdown of different strategies:

Let's explore various choices around the timing of fsync system call.

1. **After every write**: This approach involves performing fsync after writing each individual key-value pair. 
While offering the strongest durability guarantee, it incurs the highest performance penalty.
2. **After batch writes**: Transactional key-value stores often [batch](https://github.com/SarthakMakhija/go-lsm/blob/main/kv/batch.go) 
writes together. In this case, fsync would be called only upon successful transaction commit, ensuring all key-value pairs in the 
batch are written and persisted.
3. **Never (rely on OS)**: This simplest approach skips fsync entirely, relying on the operating system to handle page cache 
flushing. However, this sacrifices durability – data might be lost in case of a crash before the OS flushes the updated pages.
4. **Scheduled WAL flushes**: For key-value stores using Write-Ahead Logs (WAL), the system can periodically flush the WAL to disk, 
triggering fsync. This balances durability with performance by flushing less frequently than option 1.
5. **Memtable full flush**: If each memtable has a dedicated WAL, fsync can be performed on the WAL when the corresponding memtable 
reaches capacity. This approach combines fsync with the natural flush point of a memtable, offering a reasonable compromise between 
durability and performance.

It's important to note that only options 1 and 2 guarantee data durability since they involve explicit fsync calls.

### Recovery from WAL

During a system crash, write-ahead Logs (WALs) provide a mechanism to recover lost data. In key-value storage engines that 
maintain a dedicated WAL for each memtable, recovery involves rebuilding the memtable from its corresponding WAL.

However, recovery skips memtables that have already been flushed to SSTables (Sorted String Tables). We'll delve deeper into 
this process in the upcoming [Manifest](/en/blog/building_blocks_of_lsm_manifest/) article of this series.

#### Rebuilding Memtables from WALs

Memtable recovery entails the following steps:

1. **Read the WAL File**: The recovery process begins by opening the WAL file associated with the crashed memtable in read-only mode.
2. **Decode Key-Value Pairs**: The system iterates through the WAL file, decoding each byte sequence into individual key-value pairs. 
Decoding typically involves reading the size information of the key and value data followed by extracting the actual data bytes.
3. **Rebuild the Memtable**: For each decoded key-value pair, a callback function is invoked. This callback inserts the key-value 
pair into the rebuilt memtable data structure (likely a Skiplist).

The provided code snippet showcases the `Recover` function responsible for reading WAL:

```go
func Recover(path string, callback func(key kv.Key, value kv.Value)) (*WAL, error) {
	file, err := os.OpenFile(path, os.O_RDONLY|os.O_APPEND, 0666)
	if err != nil {
		return nil, err
	}
	bytes, err := io.ReadAll(file)
	if err != nil {
		return nil, err
	}
	for len(bytes) > 0 {
		keySize := binary.LittleEndian.Uint16(bytes)
		key := bytes[block.ReservedKeySize : uint16(block.ReservedKeySize)+keySize]

		valueSize := binary.LittleEndian.Uint16(bytes[uint16(block.ReservedKeySize)+keySize:])
		value := bytes[uint16(block.ReservedKeySize)+keySize+uint16(block.ReservedValueSize) : uint16(block.ReservedKeySize)+keySize+uint16(block.ReservedValueSize)+valueSize]

		callback(kv.DecodeFrom(key), kv.NewValue(value))
		bytes = bytes[uint16(block.ReservedKeySize)+keySize+uint16(block.ReservedValueSize)+valueSize:]
	}
	return &WAL{file: file}, nil
}
```

The provided callback function receives a decoded key-value pair and inserts it into the Skiplist of memtable.

```go
wal, err := log.Recover(log.CreateWalPathFor(id, walDirectoryPath), func(key kv.Key, value kv.Value) {
    memtable.entries.Put(key, value)
    maxTimestamp = max(maxTimestamp, key.Timestamp())
})
```

The full implementation is available [here](https://github.com/SarthakMakhija/go-lsm/blob/main/memory/memtable.go#L55).
The `Recover` method in the provided example reads the entire WAL file into memory before processing it. This raises the question 
of alternative strategies for reading and processing WAL files during recovery.

#### WAL loading and recovery strategies

1. **Full file read (current approach)**: This is the simplest approach, where the entire WAL file is loaded into memory at once. 
While straightforward, it might be inefficient for large WALs due to high memory consumption.
2. **Page-aligned WAL**: Here, WAL data is aligned to fixed sized application pages (usually multiple(s) of 4K). Recovery can then 
proceed by reading individual pages from disk. This approach improves memory usage during recovery but introduces fragmentation 
within the WAL during write operations. The author of the book [Database Design and Implementation](https://link.springer.com/book/10.1007/978-3-030-33836-7)
implements page-aligned log for his [SimpleDB](https://github.com/eatonphil/sciore-simpledb-3.4/tree/main/SimpleDB_3.4). *Please note the SimpleDB link is from Phil Eaton's git repository*.
3. **Read by encoding**:  This approach avoids reading the entire file upfront. Instead, it performs multiple, smaller file 
reads based on the data encoding. For instance, separate reads can be issued to retrieve the key size, key data, value size, and 
value data. This minimizes memory usage but involves multiple reads from the file. Apache Cassandra implements WAL using this approach.
4. **Memory-mapped files**: This approach maps the WAL file directly into memory. Reads can then be performed efficiently by 
accessing specific memory addresses. This offers good performance but requires careful memory management to avoid memory 
exhaustion, especially for large WALs. [Badger](https://github.com/dgraph-io/badger) implements WAL as memory-mapped file.

### Summary

This article provides a comprehensive overview of Write-Ahead Logs (WALs) in LSM-based key-value storage engines. 
WALs are essential for ensuring data durability and facilitating recovery from system failures.

**Key points**:

- WALs are append-only log files that store encoded key-value pairs.
- Fsync is necessary to ensure data is physically written to disk.
- Various flushing strategies balance durability and performance.
- Memtables can be recovered from WALs after system crashes.
- Multiple strategies exist for loading WALs.

### Code 

WAL implementation is available [here](https://github.com/SarthakMakhija/go-lsm/blob/main/log/wal.go).

Let's understand [SSTables](/en/blog/building_blocks_of_lsm_sstable/), the on-disk representation of data in LSM.