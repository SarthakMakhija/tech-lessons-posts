---
author: "Sarthak Makhija"
title: "Designing an LFU cache"
date: 2023-05-24
description: ""
tags: ["Cache", "TinyLFU", "Cached"]
thumbnail: /bitcask_title.webp
caption: "Background by Suzy Hazelwood on Pexels"
---

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

### Storing key/value mapping

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

