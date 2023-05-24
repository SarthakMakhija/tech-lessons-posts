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

### Making the cache memory bound

### Introducing BP-Wrapper

### Measure metrics

### Measuring cache hit ratio

### Mentions

### Code

The source code of *Cached* is available [here](https://github.com/SarthakMakhija/cached) and the crate is available [here](https://crates.io/crates/tinylfu-cached). 

### Relevant research papers

- [TinyLFU](https://dgraph.io/blog/refs/TinyLFU%20-%20A%20Highly%20Efficient%20Cache%20Admission%20Policy.pdf)
- [BP-Wrapper](https://dgraph.io/blog/refs/bp_wrapper.pdf)

### References

