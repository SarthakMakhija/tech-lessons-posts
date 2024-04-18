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
- Skip if the hash of the target key does not match the hash at the index `i` in the bucket.
- Atomically load the entry at the index `i` if the hash at that index matches the hash of the target key.
- Compare the target key and the key present in the loaded entry.
- Return if the keys match, else continue.
- Move onto the next bucket, if present, by atomically loading the next bucket pointer.

There are a few points to look at:

1. **Faster comparison using hashes**: Hash comparison (comparing `uint64` bits -> 8 bytes) is faster than comparing the actual keys, if the keys are bigger than 8 bytes. 
This is an optimization where the keys are only compared after the hash of the target key matches with the hash at an index `i`. There could be false positives (two keys may have the same hash), hence key comparison is eventually needed.
2. **No locks**: The method `Load` does not use locks, it loads the entries and the next pointer atomically. This means, the `Load` method will always see their
values either before or after the write operation by the `Store` method.
3. **Cache aligned buckets**: CLHT puts one bucket per CPU cache line. The `Load` operation loads the entire bucket (cache line) from RAM to the CPU cache(s). Since, the
entire bucket is loaded in CPU cache(s), any further comparison on entries in the bucket will not need to access RAM.

### Understanding the Store Operation

The method `Store` stores the key/value pair in the map.

```go
func (m *MapOf[K, V]) Store(key K, value V) {
	m.doCompute(
		key,
		func(V, bool) (V, bool) {
			return value, false
		},
		false,
		false,
	)
}
```

This method calls `doCompute`, so we will take a look at that method. However, `doCompute` does multiple things. So, we will take a look at it in parts.

#### Identifying the bucket

```go
table := (*mapOfTable[K, V])(atomic.LoadPointer(&m.table))
tableLen := len(table.buckets)
hash := shiftHash(m.hasher(key, table.seed))
bucketIndex := uint64(len(table.buckets)-1) & hash
```

The idea can be summarized as:

- Load the `table` atomically using `atomic.LoadPointer(&m.table)`. The field `table` is an unsafe pointer that refers to an instance of `mapOfTable`.
- Identify the `bucketIndex` where the target key has to be stored. The operation `uint64(len(table.buckets)-1) & hash` is same as `hash % number of buckets`.

> The field table is refers to an instance of `mapOfTable`. The abstraction `mapOfTable` contains a collection of buckets.
> 
> ```go
> type mapOfTable[K comparable, V any] struct {
>   //other fields omitted.
>   buckets []bucketOfPadded
> }
> ```
> 
> This field is loaded atomically in the method `doCompute` because `resize` operation might have changed the structure of the map, it might have added more buckets
> or removed buckets.

#### Acquire a lock on the identified bucket

```go
rootBucket := &table.buckets[bucketIndex]
rootBucket.mu.Lock()
```

Each bucket maintains a lock of type `sync.Mutex`. Bucket level lock is acquired in the `doCompute` method to ensure that only one goroutine performs write operations
in one bucket at one point.

#### Identify an empty entry slot

This involves finding an empty entry slot in an existing bucket. 

```go
for i := 0; i < entriesPerMapBucket; i++ {
    hashValue := atomic.LoadUint64(&currentBucket.hashes[i])
    if hashValue == uint64(0) {
        if emptyb == nil {
            emptyb = currentBucket
            emptyidx = i
        }
        continue
    }
    if hashValue != hash {
        hintNonEmpty++
        continue
    }
    entry := (*entryOf[K, V])(currentBucket.entries[i])
    hintNonEmpty++
}
```

The idea can be summarized as:

- Iterate through all entries present in the bucket. There are only three entries in each bucket.
- Check if the hashValue at index `i` is zero. Zero hash value signifies an empty entry slot.
- If the hashValue at index `i` is zero, capture the current bucket and the index. This is where the new entry will be stored, if there is no next bucket.

#### Store the key/value pair in an empty entry slot

This involves storing the key/value pair in the existing bucket at an empty entry slot. 

```go
if currentBucket.next == nil {
    if emptyb != nil {
        // Insertion into an existing bucket.
        var zeroedV V
        newValue, del := valueFn(zeroedV, false)
        
        newe := new(entryOf[K, V])
        newe.key = key
        newe.value = newValue
        
        // First we update the hash, then the entry.
        atomic.StoreUint64(&emptyb.hashes[emptyidx], hash)
        atomic.StorePointer(&emptyb.entries[emptyidx], unsafe.Pointer(newe))
        rootBucket.mu.Unlock()
        table.addSize(bucketIndex, 1)
        
        return newValue, computeOnly
    }
}	
```

The idea can be summarized as:

- Check if there is a next bucket. If there is none and an empty entry slot has been identified, write the new entry at the identified index in the existing bucket.
- Create a new instance of `entryOf` and set the key and value.
- Atomically store the hash of the key in the identified entry index using `atomic.StoreUint64(&emptyb.hashes[emptyidx], hash)`.
- Atomically store the entry pointer in the identified entry index using `atomic.StorePointer(&emptyb.entries[emptyidx], unsafe.Pointer(newe))`.
- Release the lock

#### Store in a new bucket, if all the linearly linked buckets at the idenified bucket index are full

Store the key/value pair in a new bucket and atomically link the new bucket. 

```go
if currentBucket.next == nil {
    // Insertion into a new bucket.
    var zeroedV V
    newValue, del := valueFn(zeroedV, false)
    
    // Create and append the bucket.
    newb := new(bucketOfPadded)
    newb.hashes[0] = hash
    newe := new(entryOf[K, V])
    newe.key = key
    newe.value = newValue
    newb.entries[0] = unsafe.Pointer(newe)
    atomic.StorePointer(&currentBucket.next, unsafe.Pointer(newb))
    rootBucket.mu.Unlock()
    table.addSize(bucketIndex, 1)
    return newValue, computeOnly
}
```

The idea can be summarized as:

- Check if there is a next bucket. If there is none and no empty slot has been identified, create a new bucket.
- Create a new instance of `entryOf` and set the key and value.
- Store the new entry in the 0th index of the newly created bucket.
- Atomically link the new bucket using `atomic.StorePointer(&currentBucket.next, unsafe.Pointer(newb))`
- Release the lock

The complete method is available [here](https://github.com/puzpuzpuz/xsync/blob/main/mapof.go#L258).

The `Store` operation also checks if a `resize` operation is in progress.

```go
if m.resizeInProgress() {
    // Resize is in progress. Wait, then go for another attempt.
    rootBucket.mu.Unlock()
    m.waitForResize()
    goto compute_attempt
}

if m.newerTableExists(table) {
    // Someone resized the table. Go for another attempt.
    rootBucket.mu.Unlock()
    goto compute_attempt
}
```

It considers the following:
- If a resize operation is in progress, the `Store` operation waits for `resize` to finish and starts over. 
- If `resize` is not in progress but someone resized the map after the `Store` operation was invoked, then release the lock and start again.

There are a few points to look at:

1. Each bucket maintains a lock `sync.Mutex`. Bucket level lock is acquired in the `doCompute` method. This means a single goroutine case perform write operations
in one bucket.
2. If the number of goroutines that perform the write operations on `MapOf` are too many (say 9000), then larger number of buckets should help because goroutine
contention will be reduced. The number of buckets should be determined based on the benchmarks.
3. The method `Store` performs atomic operation on `next` pointer, `entries` and `hashes` of a bucket. Since, these are atomic operations, none of the goroutines
will see an inconsistent value for any of these individual fields.

> Consider two goroutines, one is doing the `Store` operation and the other is doing the `Load` operation on the same key that the store goroutine is trying to put. 
>
> The goroutine performing the `Store` operation finds an empty
> slot and updates the hash and then the entry by executing `atomic.StoreUint64(&emptyb.hashes[emptyidx], hash)` and `atomic.StorePointer(&emptyb.entries[emptyidx], unsafe.Pointer(newe))`.
>
> It is possible for the goroutine performing the `Load` operation to see the updated hash because the hash is atomically written by the store goroutine at an index `i`, but the value is not yet 
> written. So, the load goroutine will return an empty value which will still be the expected result.

> Cache line update ....

### Understanding the Resize Operation

### References

- [xsync](https://github.com/puzpuzpuz/xsync)
- [Designing ASCY-compliant Concurrent Search Data Structures](https://infoscience.epfl.ch/record/203822)
