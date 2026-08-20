---
title: "Rhashtables: under the hood"
url: https://lwn.net/Articles/751974/
date: "April 21, 2018"
category: Hash table
author: "April 21, 2018 This article was contributed by Neil Brown"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

April 21, 2018

This article was contributed by Neil Brown

[The first article in this series](<https://lwn.net/Articles/751374/>) described the interface to the "rhashtable" resizable hash-table abstraction in Linux 4.15. While a knowledge of the interface can result in successful use of rhashtables, it often helps to understand what is going on "under the hood", particularly when those details leak out through the interface, as is occasionally the case with rhashtable. The centerpiece for understanding the implementation is knowing exactly how the table is resized. So this follow-on article will explain that operation; it will also present the configuration parameters that were skimmed over last time and discuss how they affect the implementation. 

#### The chain gang — tables within tables

An rhashtable table ([`struct rhashtable`](<https://elixir.bootlin.com/linux/v4.15/source/include/linux/rhashtable.h#L152>)) contains a list of bucket tables ([`struct bucket_table`](<https://elixir.bootlin.com/linux/v4.15/source/include/linux/rhashtable.h#L76>)). Normally there is just one bucket table, or two when the table is being resized. In general there can be more, though making this happen in practice would require adding many objects more quickly than the table can be "rehashed", which is the processes of moving objects from the early bucket tables to the last. 

Each bucket table has an array of [`struct rhash_head`](<https://elixir.bootlin.com/linux/v4.15/source/include/linux/rhashtable.h#L66>) buckets for the various hash chains and a separate array of spinlocks, each of which protects two or more of the chains during modification. The different insert, lookup, and remove operations act on each bucket table in turn, stopping when they find what they are looking for or when they get to the last bucket table. Insertion will only insert into that last table, if anywhere. 

There are a few different times when a new bucket table is added to the chain. When an insert function finds that the total number of objects in the table exceeds 70% of the number of buckets in the last bucket table in the chain, it schedules an asynchronous worker thread to look into this situation. Similarly, when object removal finds that, in a table where automatic shrinking is enabled, the number of objects is less than 30% of the number of buckets, the worker is again scheduled. This worker repeats the checks and possibly allocates a bucket table that is either double or half the previous size. If there is more than one bucket table at that point, the worker will proceed to move all objects to the last. 

If insertion ever finds that the number of objects exceeds 100% of the bucket count it assumes that the thread is taking too long, so it takes matters into its own hands and allocates a new bucket table directly. Since this must happen without blocking (none of the rhashtable interfaces except `rhashtable_init()` and `rhashtable_destroy()` ever block), that allocation can fail. If it doesn't fail, the new object is inserted in the new table and, hopefully, the rehashing worker thread will eventually catch up. Insertion will also trigger an immediate allocation if it finds that the bucket it wanted to insert into already has 16 objects. Since the table is resized to keep the average bucket occupancy below one, this suggests that something has gone wrong with the hash function — possibly an attacker has found a way to generate lots of objects with the same hash. 

One of the per-bucket-table values is a random hash seed (`hash_rnd`) that is passed to the hash function to be included in the calculated hash. When a new bucket table is allocated, it will have a new `hash_rnd`, so objects will be placed in different chains. This should mean that the 17 objects that all had the same hash will now be more evenly distributed. This is a significant difference from the original "relativistic hash table" implementation, which assumed that the hash of an given object remained unchanged; the current implementation deliberately changes it for each new bucket table. 

#### Performing the rehash

![\[Moving an
object to a new table\]](https://static.lwn.net/images/2018/rhashtable.png)

Whenever a new bucket table is allocated, whether to change the size of the table or just to change the hash seed, every object in the old table must be moved to the new table. This process seems labor-intensive, but is pleasantly simple. Since it happens rarely and is performed asynchronously with respect to all other accesses, the cost is unlikely to be a concern. 

At a simple level, the rehash process takes one object from the old table and inserts it in the new table, then repeats that until no objects are left. This can only be correct if an asynchronous lookup for that object, which proceeds with no locking, will always be certain of finding the object. To provide this correctness, rhashtable always moves the _last_ object from a hash chain, and attaches it to the start of the destination hash chain. It attaches the object to the new chain _before_ removing it from the old chain; see the diagram to the right. In pseudocode, the full rehash sequence is: 

```
for each bucket in the first bucket table
            take a lock on the current bucket
        	while the current bucket is not empty
                find the last object in the bucket
                determine the destination bucket in the last bucket table
                lock the destination bucket
                attach the object to the head of the new bucket
                remove the object from the tail of the old bucket
           	    unlock the new bucket
            unlock the current bucket
        detach the first bucket table
```

This insert-before-remove sequence means there is never a time when the object is not in the table, so a lookup will never fail to find the object. It also means that, for a short time, a chain in the old table is connected to a chain in the new table. Having joined chains will not hurt lookup as it already expects to see unwanted objects in the chain, so a few more cannot hurt. It also cannot hurt insert or removal as they take bucket locks that the rehash process also takes, so they will never see the joined chain. It can affect a `rhashtable_walk_next()` though. 

#### Walking the hash table while the table is being rehashed

As mentioned in the previous article, it is not possible to guarantee that a walk of the hash table will see objects at most once if a rehash could happen while the walk is proceeding. One reason is that, as we have just seen, the "current" object might move from one bucket table to another and be attached to the start of a hash chain. If that hash chain was not empty, then the objects already there might have been moved from a hash chain that has already been walked along. Since hash chains are kept short this is quite unlikely, but it is a real possibility. 

When `rhashtable_walk_next()` completes the last hash chain in a bucket table, it checks if there is another in the chain of tables. Normally, there is not and `rhashtable_walk_next()` will return `NULL` to indicate completion. If there is, `rhashtable_walk_next()` will return `‑EAGAIN` and prepare to walk through the next table. When a caller receives `‑EAGAIN`, it will know that a rehash is underway and, thus, it might have already seen a duplicate. It can choose to give up and return an error or incomplete data, or it can keep going and expect to see every object in the table again, including any that it might have missed the first time as they might have been rehashed before the walker saw them. 

If a caller needs to be aware of rehashing, and is using `rhashtable_walk_stop()` and `rhashtable_walk_start()` to drop the RCU read lock, it should use `rhashtable_walk_start_check()` instead of `rhashtable_walk_start()`. This alternate interface will report `‑EAGAIN` if the bucket table that was being walked through was discarded while the lock wasn't held. This has exactly the same implications as `rhashtable_walk_next()` returning `‑EAGAIN` in that subsequent calls will start walking the next bucket table. 

The only way to guarantee that a walk sees all objects exactly once would be to disable the rehash process. There is no documented interface for doing this, but in the current implementation it can be done by claiming the hash table mutex: 

```
mutex_lock(&my_objects.mutex);
```

This call will wait for any current rehash to complete, and will stop future rehashes from starting. If lots of objects are added while the mutex is held, the insert function will eventually allocate a new bucket table and will start inserting into that. If this memory allocation fails, the insertion will fail. This is one of a small number of ways that insertion can fail. 

#### Failure during insertion

It must be expected that insertion into the hash table will fail if the key is already in use, unless of course you use `rhashtable_insert_fast()`, which doesn't check. There are a few other circumstances that can trigger errors; three error codes (beyond `‑EEXIST`) are possible: 

  * `‑ENOMEM` is returned when the insert function tries to allocate a bucket table and fails. If the required table is larger than one page, it will try to fall back on allocating a single page and use that to hold pointers to other pages to be allocated later. If this works we might not get `‑ENOMEM` immediately but could still get it if a subsequent single-page allocation fails. 
  * `‑EBUSY` is returned if the insert function finds that there is more than one bucket table and the last one has 16 objects already in the hash chain. This is almost impossibly unlikely, unless the rhashtable is using a custom hash function that is ignoring the random seed or doing something else strange. 
  * `‑E2BIG` is returned if there are already 231 objects in the hash table. This prevents the element counter from overflowing. `‑E2BIG` can also be returned if a maximum table size was specified when the table was created (as described below), and the number of objects has reached twice that maximum size. 

#### The full scoop on configuration parameters

Now that we know how rehashing works, and how it affects walking the table, it remains only to see how it can be configured through values in the [`rhashtable_params`](<https://elixir.bootlin.com/linux/v4.15/source/include/linux/rhashtable.h#L122>) structure. Of the twelve fields in this structure we have already met six (`key_len`, `key_offset`, `head_offset`, `hashfn`, `obj_cmpfn`, and `obj_hashfn`) and five are related to configuring the size for rehashing. The final parameter is `nulls_base`. It is essentially unused and unusable in the current implementation, so it is best ignored. 

The size-related parameters are: 

  * `u16 nelem_hint;`

`nelem_hint` is documented as a hint regarding the number of elements that the table will have, but is never really used that way; in fact I cannot imagine how anyone would be able to make a meaningful guess. Some callers set it to two or three with the intention to "use a really small initial bucket table" — both of these values result in four buckets being allocated. 

Other callers use values like 192, 384, or 768 which mean "use a bigger initial table, maybe 256, 512, or 1024 buckets". I can understand memory conscious coders asking for a small initial number of buckets if they don't think the table will be used much, though the default of 64 buckets won't often be a burden. It is harder to understand why people would use a resizable hash table, but not just give it complete control of the size. 
  * `unsigned int max_size;`

The `max_size` is rounded down to a power of two, then used as a maximum size for the bucket table and half the maximum number of objects in the table. This maximum number of objects is a hard limit; insertion will fail when it is reached. The default maximum is 231. 
  * `bool automatic_shrinking;`

If this flag is set, the table will be resized downwards (halving the size) when the residency drops below 30% (objects per bucket). Resizing upward is always enabled, resizing downward must be explicitly requested as in some cases it might be wasteful effort. 
  * `unsigned int min_size;`

`min_size` is rounded down to a power of two but kept to at least four (the default). This puts a lower bound on resizing the bucket table. When `automatic_shrinking` is not set, this value is ignored; it does not, for example, impose a minimum on the initial allocation. 
  * `u8 locks_mul;`

As mentioned, each bucket table has a set of locks protecting the buckets: two or more buckets per lock. On machines with few CPUs, even having only half as many locks as buckets can be wasteful when there are a great many buckets. To avoid wasting memory on useless locks, rhashtable limits the number of locks using the following formula: 

```
num_locks = min(num_chains/2, locks_mul * min(num_possible_cpus, 64))
```

If `locks_mul` is not given, `32` is used, so by default my eight-CPU desktop would get at most 256 locks for each bucket table. The only examples of setting this field in the current kernel involve setting to it to one, which is likely to be plenty for all but the busiest tables. 

That completes the set of parameters that affect resizing. My recommendation would be to not set any of them unless you feel that memory conservation is more important than raw speed, in which case set `automatic_shrinking` to `true` and `locks_mul` to `1`. 

#### A word of warning

It is nice to have rhashtables properly documented, but I wouldn't expect that situation to last. While working on this article, I have been discussing possible changes with the maintainers. None of these are major and should not invalidate this documentation, but they are unlikely to be the last changes ever suggested. Rhashtables are quite different from what they were four years ago, and may be different again in four more years — the Linux kernel is ever a dynamic place. So please don't trust this description implicitly — check it against the code, just in case.
