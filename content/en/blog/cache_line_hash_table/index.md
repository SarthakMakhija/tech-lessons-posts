---
author: "Sarthak Makhija"
title: "Cache-Line Hash Table"
date: 2024-04-18
description: "
"
tags: ["CPU Cache-Line", "Hash Table", "CLHT", "Cache-Line Hash Table", "xsync"]
thumbnail: 
caption: ""
---

### Introduction

### Understanding CPU Cache line

### Understanding CLHT (Cache-Line Hash table)

CLHT stands for cache-line hash table as it tries to put one bucket per CPU cache line. The core idea behind the design of CLHT is to minimize the amount cache coherence
traffic. In concurrent data structures, cache coherence traffic is generated when a thread running on a core updates a cache line while the other remote 
cores hold the same cache line. In this scenario, cache coherence protocol would kick in, thus requiring the other cores to invalidate the cache line and fetch it from
RAM. The core ideas behind CLHT include:
- Minimize the cache coherence traffic by reducing the number of cache lines that are written in an update/put operation
- Perform in-place update of key/value pairs
- Take no locks during read (/get) operations

I will be taking the example of the [MapOf](https://github.com/puzpuzpuz/xsync/blob/main/mapof.go) implementation from [xsync](https://github.com/puzpuzpuz/xsync/) which is a CLHT-inspired 
concurrent hash map.

The core abstraction `MapOf` contains an [unsafe pointer](https://pkg.go.dev/unsafe#Pointer) to the `mapOfTable` struct. 

```go
type MapOf[K comparable, V any] struct {
	//other fields omitted.
	table        unsafe.Pointer // *mapOfTable
}
```

The abstraction `mapOfTable` contains a slice of buckets. The buckets are linearly linked using the `next` field at each index. CLHT puts one bucket per CPU cache line,
this simply means that the size of each bucket should be equal to the [CPU cache line size](https://docs.rs/crossbeam-utils/latest/crossbeam_utils/struct.CachePadded.html).
With `entriesPerMapBucket` as 3, the size of a single instance of `bucketOf` is 64 bytes, which is the size of CPU cache line on most of the processors. 
Each instance of `bucketOf` contains three hashes and three entries. Each entry is an unsafe pointer to the `entryOf` struct which contains a key/value pair.

```go
const entriesPerMapBucket = 3

type mapOfTable[K comparable, V any] struct {
    //other fields omitted.
	buckets []bucketOfPadded
}

type bucketOfPadded struct {
    pad [cacheLineSize - unsafe.Sizeof(bucketOf{})]byte
    bucketOf
}

type bucketOf struct {
    hashes  [entriesPerMapBucket]uint64
    entries [entriesPerMapBucket]unsafe.Pointer // *entryOf
    next    unsafe.Pointer                      // *bucketOfPadded
    mu      sync.Mutex
}

type entryOf[K comparable, V any] struct {
    key   K
    value V
}
```

> The abstraction `MapOf` uses unsafe pointers which are later used in atomic pointer operations, like `atomic.StorePointer(&b.entries[i], unsafe.Pointer(newEntry))`.

The design of xsync `MapOf` is presented in the below image.

<div class="align-center-exclude-width-change">
    <img src="/xsync.png" alt="Design of xsync MapOf"/>
</div>

> An unsafe pointer represents a pointer an arbitrary type.
> Let's take a quick example of converting a byte slice to string using unsafe.
> ```go
> func toString(buffer []byte) string {
>    unsafePointer := unsafe.Pointer(&buffer)
>    stringPointer := (*string)(unsafePointer)
>    
>	return *stringPointer
>}
> ```
>
> This snippet creates an unsafe pointer using the address of the byte slice, casts it to a string pointer and then dereferences it to get string. This works because
> both the byte slice and the string share an equivalent memory layout and the resulting string is not larger than the input byte slice.

### Understanding the Load Operation

The `Load` operation returns the value if the target key is present, else returns a zero value of type V.

```go
func (m *MapOf[K, V]) Load(key K) (value V, ok bool) {
	table := (*mapOfTable[K, V])(atomic.LoadPointer(&m.table))
	hash := shiftHash(m.hasher(key, table.seed))
	bucketIndex := uint64(len(table.buckets)-1) & hash
	rootBucket := &table.buckets[bucketIndex]
	
	for {
		for i := 0; i < entriesPerMapBucket; i++ {
			h := atomic.LoadUint64(&rootBucket.hashes[i])
			if h == uint64(0) || h != hash {
				continue
			}
			entryPointer := atomic.LoadPointer(&rootBucket.entries[i])
			if entryPointer == nil {
				continue
			}
			entry := (*entryOf[K, V])(entryPointer)
			if entry.key == key {
				return entry.value, true
			}
		}
		bucketPointer := atomic.LoadPointer(&rootBucket.next)
		if bucketPointer == nil {
			return
		}
		rootBucket = (*bucketOfPadded)(bucketPointer)
	}
}
```

The idea behind the `Load` operation can be summarized as:

- Identify the `bucketIndex` where the target key may be present. The operation `uint64(len(table.buckets)-1) & hash` is same as `hash % number of buckets`.
- Iterate through all entries present in the bucket. There are only three entries in each bucket.
- Skip if the hash of the target key does not match the hash of the entry at the index `i` in the bucket.
- Atomically load the entry at the index `i` if the hash at that index matches the hash of the target key.
- Compare the target key and the key present in the loaded entry.
- Return if the keys match, else continue.
- Move onto the next bucket, if present, by atomically loading the next bucket pointer.

There are a few points to look at:

1. **Faster comparison using hashes**: Hash comparison (comparing `uint64` bits -> 8 bytes) is faster than comparing the actual keys, if the keys are bigger than 8 bytes. 
This is an optimization where the keys are only compared after the hash of the target key matches with the hash at an index `i`. There could be false positives (two keys may have the same hash), hence key comparison is eventually needed.
2. **No locks**: The method `Load` does not use locks, it loads the entries and the next pointer atomically. This means, the `Load` method will always see their
values either before or after the write operation by the `Store` method.
3. **Cache aligned buckets**: CLHT puts one bucket per CPU cache line. The `Load` oepration loads the entire bucket (cache line) from RAM to the CPU cache(s). Since, the
entire bucket is loaded in CPU cache(s), futher comparison on entries in the bucket will not need to access RAM.

### Understanding the Store Operation


### Understanding the Resize Operation

### References

- [xsync](https://github.com/puzpuzpuz/xsync)
- [Designing ASCY-compliant Concurrent Search Data Structures](https://infoscience.epfl.ch/record/203822)
