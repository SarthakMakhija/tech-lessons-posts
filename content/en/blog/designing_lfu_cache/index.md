---
author: "Sarthak Makhija"
title: "Designing an LFU cache"
date: 2023-05-24
description: ""
tags: ["Cache", "TinyLFU", "Cached"]
thumbnail: /bitcask_title.webp
caption: "Background by Suzy Hazelwood on Pexels"
---

https://en.wikipedia.org/wiki/Least_frequently_used

https://dgraph.io/blog/post/introducing-ristretto-high-perf-go-cache/

### Measuring access frequency

All LFU caches need to maintain the access frequency for each key.
Storing the access frequency in a `HashMap` like data structure would mean that the space used to store the frequency is directly proportional to the number of keys in the cache.
This is an opportunity to use a probabilistic data structure like [count-min sketch](https://tech-lessons.in/blog/count_min_sketch/) and make a trade-off between the accuracy of the access frequency and the space used to store the frequency.  

> Count-min sketch (CM sketch) is a probabilistic data structure used to estimate the frequency of events in a data stream.
> It relies on hash functions to map events to frequencies, but unlike a hash table, it uses only **sublinear space** at the expense of over-counting some events due to hash collisions. The count–min sketch was invented in 2003 by Graham Cormode and S. Muthu Muthukrishnan.

`Cached` uses count-min sketch inside the abstraction [FrequencyCounter](https://github.com/SarthakMakhija/cached/blob/main/src/cache/lfu/frequency_counter.rs) to store the frequency for each key.

Count-min sketch is represented as a D*W matrix, where D is the total number of hash functions (or depth) and W is the width or the number of counters per hash function. 

<div class="align-center">
    <img src="/countminsketch.png"/>
</div>

The matrix is initialized with zero at the beginning. A count-min sketch can be represented with the following structure:

```rust
#[repr(transparent)]
#[derive(Debug, PartialEq)]
struct Row(Vec<u8>);

const ROWS: usize = 4;

pub(crate) struct FrequencyCounter {
    matrix: [Row; ROWS],
    seeds: [u64; ROWS],
}

//initialize the matrix
fn matrix(total_counters: TotalCounters) -> [Row; ROWS] {
    let total_counters = (total_counters / 2) as usize;
    let rows =
        (0..ROWS)
            .map(|_index| Row(vec![0; total_counters]))
            .collect::<Vec<Row>>();

    rows.try_into().unwrap()
}
```

We still need to make a decision on the number of counters. What should be the number of counters to get the closest estimate on the access frequency?

We can start with a simple theory. If an LFU cache is going to contain N keys, then we can keep N counters in the count-min sketch. However, we need to consider
hash conflicts. 

Let's understand the logic of increment with the following pseudocode.

```rust
    pub(crate) fn increment() {
        // 1) Iterate through all the rows (row = 0 to depth = D)
        // 2) Get the hash of the incoming key
        // 3) Perform `hash % self.total_counters` to identify the column index (width = W)
        // 4) Increment the value at the identified column in the row R(i)
    }
```

Keeping the counters same as the number of keys that would be contained in the cache would mean a higher error rate in the access frequency estimate (because of hash conflicts).

To keep the estimates from wavering (/overestimating) too much because of hash conflicts, we need to have `counters = K times the number of keys`. Any choice of K is an attempt at reducing the hash conflict
of keys in each row. 

`Cached` proposes `K = 10` and does [performance benchmarks](https://github.com/SarthakMakhija/cached/blob/main/benches/benchmarks/frequency_counter.rs) with `K = 2` and `K = 10`.

Let's understand a way store the key/value mapping.

### Storing key/value mapping

We need a way to store the value by key. This is done by the [Store](https://github.com/SarthakMakhija/cached/blob/main/src/cache/store/mod.rs) abstraction in Cached.
`Store` uses [DashMap](https://docs.rs/dashmap/latest/dashmap/) which is a concurrent associative array/hashmap. 

`DashMap` maintains an array named `shards` and each element is a `RwLock` to a `HashMap`. The `put` operation identifies the `shard_index`, acquires a `write lock` 
against that shard and writes to the `HashMap`.

```rust
pub struct DashMap<K, V, S = RandomState> {
    shards: Box<[RwLock<HashMap<K, V, S>>]>,
    //code omitted
}
```
The following code represents `Store`.

```rust
pub(crate) struct Store<Key, Value>
    where Key: Hash + Eq, {
    store: DashMap<Key, StoredValue<Value>>,
}

pub struct StoredValue<Value> {
    value: Value,
    key_id: KeyId,
    expire_after: Option<ExpireAfter>,
    pub(crate) is_soft_deleted: bool,
}
```

There is another decision to be made here. 

How should we return the value to the clients? 

- Option1: return a reference of the value
- Option2: force the clients to provide a cloneable value as a part of the `put` operation and return the `Some<Value>` by cloning the value, if it exists for the key
- Option3: provide both the options

Let's look at Option1 first. In order to return a reference to the value we need to look into the `get` method of `DashMap`.

```rust
pub fn get<Q>(&'a self, key: &Q) -> Option<Ref<'a, K, V, S>>
where
    K: Borrow<Q>,
    Q: Hash + Eq + ?Sized, { self._get(key) }

pub struct Ref<'a, K, V, S = RandomState> {
    _guard: RwLockReadGuard<'a, HashMap<K, V, S>>,
    k: *const K,
    v: *const V,
}

pub type RwLockReadGuard<'a, T> = lock_api::RwLockReadGuard<'a, RawRwLock, T>;

//part of lock_api
pub struct RwLockReadGuard<'a, R: RawRwLock, T: ?Sized> {
    rwlock: &'a RwLock<R, T>,
    marker: PhantomData<(&'a T, R::GuardMarker)>,
}
```

- The `get` method of `DashMap` returns an `Option<Ref<'a, K, V, S>>`. 
- `Ref` is a publicly available struct and has a lifetime annotation `'a` which is tied to the lifetime of `RwLockReadGuard`.
- The lifetime of `RwLockReadGuard` of `DashMap` is tied to lifetime of `lock_api::RwLockReadGuard`. 
- The lifetime of `RwLockReadGuard` is tied to the lifetime of `RwLock`

This means if we decide to return a reference to the value for a key we are actually returning `DashMap's Ref` and also holding a lock against the shard that the key belongs to.

The [Store](https://github.com/SarthakMakhija/cached/blob/main/src/cache/store/mod.rs) abstraction in `Cached` provides `get_ref` method that returns an instance of [KeyValueRef](https://github.com/SarthakMakhija/cached/blob/main/src/cache/store/key_value_ref.rs) which wraps `DashMap's Ref`.

```rust
pub(crate) fn get_ref(&self, key: &Key) -> Option<KeyValueRef<'_, Key, StoredValue<Value>>> {...}

pub struct KeyValueRef<'a, Key, Value>
    where Key: Eq + Hash {
    key_value_ref: Ref<'a, Key, Value>,
}
```

At the same time, `Store` provides `get` method if the value is cloneable, which returns an `Option<Value>`. This behavior will cause `DashMap` to hold the lock against the shard
while the value is being read, clones the value, returns the value to the client, and drops the lock.

```rust
impl<Key, Value> Store<Key, Value>
    where Key: Hash + Eq,
          Value: Clone, {
    pub(crate) fn get(&self, key: &Key) -> Option<Value> {
        let maybe_value = self.store.get(key);
        let mapped_value = maybe_value
            .filter(|stored_value| stored_value.is_alive(&self.clock))
            .map(|key_value_ref| { key_value_ref.value().value() }); //clone the value

        mapped_value
    }
}
```

Let's bring another requirement: "memory bound cache".

### Making the cache memory bound

We want to design a cache that uses a fixed amount of memory determined by some configuration parameter. 

This requirement brings in two concepts:

1) Every key/value pair should have some *weight*, and we should be able to determine the total weight used by the cache.
2) We must ensure that the total cache weight does not cross the specified limit.

> `Cached` uses the term "weight" to denote the space (/size).

Let's understand the first point.

In order to associate weight with each key/value pair, we can provide a method takes `weight` as a parameter along with `key` and `value`.
[Cached](https://github.com/SarthakMakhija/cached/blob/main/src/cache/cached.rs) provides a method `put_with_weight()` that allows the clients to specify the weight associated with each key/value pair.

Other option is to auto-calculate the weight for each key/value pair. In order to calculate the weight, we should be able to calculate the size of each key/value pair
using functions like `std::mem::size_of_val()` or `std::mem::size_of()` and add to that the size of any additional metadata, like `expiry: Duration`, that might be stored. 

The weight calculation in `Cached` is available [here](https://github.com/SarthakMakhija/cached/blob/main/src/cache/config/weight_calculation.rs).

Now that we have calculated the weight of each key/value pair, we should be able to maintain the weight of each key and the total weight used by the cache.
`Cached` uses [CacheWeight](https://github.com/SarthakMakhija/cached/blob/main/src/cache/policy/cache_weight.rs) as an abstraction to maintain the weight of each key in the cache, and it also manages the total weight used by the cache. 
The weight of each key is maintained via `WeightedKey` abstraction that contains the `key`, `key_hash` and its `weight`.
Every `put` results in increasing the *cache weight*, every delete results in reducing the *cache weight* and every update probably results in changing in the *cache weight*.

```rust
pub(crate) struct CacheWeight<Key>
    where Key: Hash + Eq + Send + Sync + Clone + 'static, {
    max_weight: Weight,
    weight_used: RwLock<Weight>,
    key_weights: DashMap<KeyId, WeightedKey<Key>>,
}
```

We now have a weight associated with each key/value pair.

### Admission and eviction policy

Our cache is a memory bound cache and this poses an interesting challenge. 

*Should we admit the incoming key/value pair after the cache has reached its weight? If yes, which keys should be evicted to create the space because we can not let 
the total cache weight increase beyond some threshold?*

This is where the paper [TinyLFU](https://dgraph.io/blog/refs/TinyLFU%20-%20A%20Highly%20Efficient%20Cache%20Admission%20Policy.pdf) comes into the picture. 
The main idea is to only let in a new key/value pair if its access estimate is higher than that of the item being evicted. This simply means that the incoming
key/value pair should be more valuable to the cache than some existing key/value pairs. 

Let's look at the approach:

**Approach**:
1) Get an estimate of the access frequency of the incoming key. The incoming key has not been accessed yet, but we *might* get some access count
   because of hash conflicts since we rely on the probabilistic data structure [count-min sketch](https://tech-lessons.in/blog/count_min_sketch/)
   to maintain the access frequencies of the keys.
2) What we are trying to do is get the approximate estimation of the access frequency if this key were admitted in the cache
3) Get a sample of the existing keys that consist of `keyId`, its `weight` and `its access frequency`. *Sample size can be configurable.*
4) Pick the key with the smallest access frequency from the sample (K1)
5) If the access frequency of the incoming key is less than the access frequency of the key K1, then reject the incoming key because
   its access frequency is less than the smallest access frequency in the sample.
6) Else, `delete` the key `K1` and create the space in the cache. The space created will be equal to the `weight of K1`.
7) Repeat the process until either the incoming key is rejected or enough space to accommodate the incoming key is created in the cache.

This approach is called "Sampled LFU".
`Cached` uses [AdmissionPolicy](https://github.com/SarthakMakhija/cached/blob/main/src/cache/policy/admission_policy.rs) to decide whether an incoming key/value pair should be admitted.

There is still one more case to consider. What if there is a key with high access frequency, and it has not been seen for a while. Will it never get evicted?

This point is around the recency of a key access and [TinyLFU](https://github.com/SarthakMakhija/cached/blob/main/src/cache/lfu/tiny_lfu.rs) ensures the recency of key access by a `reset` method. 
We have used `count-min sketch` to maintain the access frequency of each key and everytime a key is accessed, the frequency counter is incremented.
After N key increments, the counters get halved. So, a key that has not been seen for a while would also have its counter reset to half of the original value; 
thereby providing a chance to the new incoming keys to get in.

>> [TinyLFU](https://github.com/SarthakMakhija/cached/blob/main/src/cache/lfu/tiny_lfu.rs) has a [FrequencyCounter](https://github.com/SarthakMakhija/cached/blob/main/src/cache/lfu/frequency_counter.rs) and a [DoorKeeper](https://github.com/SarthakMakhija/cached/blob/main/src/cache/lfu/doorkeeper.rs).
>> DoorKeeper is implemented using a [Bloom filter](https://tech-lessons.in/blog/bloom_filter/). Before increasing the access frequency of a key in `FrequencyCounter` via
>> `TinyLFU`, a check is done in the `DoorKeeper` to see if the key is present. Only if the key is present in the `DoorKeeper`, its access count is incremented.
>> This is to ensure that `FrequencyCounter` does not end up having a long tail of keys that are not seen more than once. 

<<We now have a,b,c...>>

Let's understand contention.

### Introducing BP-Wrapper

We have already seen `Store` and `count-min sketch` based `FrequencyCounter`. Technically, both these data structures are shared data structures and prone to [contention](https://stackoverflow.com/questions/1970345/what-is-thread-contention).

Let's understand this with an example. Imagine there is a `get` request for an existing key "topic". The key has been accessed, so we should to increment the access counter for the key.
`FrequencyCounter` is a shared data structure, so an option to increment the access counter is to take a lock on the entire data structure and then increment the 
count. If multiple threads are trying to increase the access count for same or different keys, all these threads would be contending for a single write lock on `FrequencyCounter`.

This is where the paper [BP-Wrapper](https://dgraph.io/blog/refs/bp_wrapper.pdf) comes in. This paper suggests two ways of dealing with contention *prefetching* and *batching*.

`Cached` uses *batching* both with `get` and `put` operations. 

#### Get

A `get` operation returns the value for a key if it exists. It queries the `Store` and gets the value. The next step is to increase the access count for the key. This is where
the idea of *batching* comes in. All the *gets* are batched in a ring-buffer like abstraction called [Pool](https://github.com/SarthakMakhija/cached/blob/main/src/cache/pool.rs). `Pool`
is a collection of [Buffer](https://github.com/SarthakMakhija/cached/blob/main/src/cache/pool.rs) and each `Buffer` is a collection of hashes of the keys.

Any time a key is accessed, it is added to a buffer within the `Pool`. When a buffer is full, it is drained. Draining involves sending the entire `Vec<KeyHash>` to a [BufferConsumer](https://github.com/SarthakMakhija/cached/blob/main/src/cache/buffer_event.rs).
`BufferConsumer` is implemented using a single thread that receives the buffer content from a channel. The `BufferConsumer` on receiving the buffer content applies it to the `FrequencyCounter`, there by incrementing the frequency access of the keys.
The channel size is kept small to further reduce contention and the memory footprint of collecting the buffer in memory. If the channel is full, the buffer content is dropped.

[AdmissionPolicy](https://github.com/SarthakMakhija/cached/blob/main/src/cache/policy/admission_policy.rs) plays the role of `BufferConsumer` in `Cached`.

>> The current implementation of `Cached` uses fine-grained lock over each buffer: `buffers: Vec<RwLock<Buffer<Consumer>>>`. The next release may change this implementation.

#### Put

The idea with `get` was to buffer the accesses and apply them to the `FrequencyCounter` when the buffer gets full. We can not apply the same idea with `put` because we want
to serve the `put` operations as soon as possible. However, the idea of batching is still relevant with `put` (or any other write) operations. 

Treat every write operation (`put`, `update`, `delete`) as a command, send to a [mpsc channel](https://doc.rust-lang.org/std/sync/mpsc/) and have a single thread receive
commands from the channel and execute then one after the other.

`Cached` follows the same idea. Every write operation goes as a [Command](https://github.com/SarthakMakhija/cached/blob/main/src/cache/command/mod.rs) to a 
[CommandExecutor](https://github.com/SarthakMakhija/cached/blob/main/src/cache/command/command_executor.rs). `CommandExecutor` is implemented as a single thread
that receives commands from a [crossbeam-channel](https://crates.io/crates/crossbeam-channel). Every time a command is received, it is executed by this single thread of execution.

This design choice has obvious implications. 
- There is no guarantee that a `put` operation will happen successfully because it may be rejected by the `AdmissionPolicy` as described in the section [Admission and eviction policy](#admission-and-eviction-policy)
- All the write operations are processed at a later point in time

The solution to both these points lie in providing the right feedback to the clients. `Cached` provides [CommandAcknowledgementHandle](https://github.com/SarthakMakhija/cached/blob/main/src/cache/command/acknowledgement.rs) as an abstraction that implements the [Future](https://doc.rust-lang.org/std/future/trait.Future.html) trait.
Every write operation is returned an instance of [CommandSendResult](https://github.com/SarthakMakhija/cached/blob/main/src/cache/command/command_executor.rs) that wraps 
`CommandAcknowledgement` which provides a `handle()` method to return a `CommandAcknowledgementHandle` to the clients. 

Clients can perform `await` on the `CommandAcknowledgementHandle` and get the [CommandStatus](https://github.com/SarthakMakhija/cached/blob/main/src/cache/command/mod.rs).

```rust
#[tokio::main]
 async fn main() {
    let cached = CacheD::new(ConfigBuilder::new(100, 10, 100).build());
    let status = cached.put("topic", "microservices").unwrap().handle().await;
    assert_eq!(CommandStatus::Accepted, status);
}
```

### Measure metrics

### Measuring cache hit ratio

### Mentions

### Code

The source code of *Cached* is available [here](https://github.com/SarthakMakhija/cached) and the crate is available [here](https://crates.io/crates/tinylfu-cached). 

### Relevant research papers

- [TinyLFU](https://dgraph.io/blog/refs/TinyLFU%20-%20A%20Highly%20Efficient%20Cache%20Admission%20Policy.pdf)
- [BP-Wrapper](https://dgraph.io/blog/refs/bp_wrapper.pdf)

### References

