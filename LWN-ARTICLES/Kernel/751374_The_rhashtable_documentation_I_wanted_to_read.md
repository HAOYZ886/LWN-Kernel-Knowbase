---
title: The rhashtable documentation I wanted to read
url: https://lwn.net/Articles/751374/
date: "April 13, 2018"
category: Hash table
author: "April 13, 2018 This article was contributed by Neil Brown"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

April 13, 2018

This article was contributed by Neil Brown

The rhashtable data structure is a generic resizable hash-table implementation in the Linux kernel, which LWN first [introduced](<https://lwn.net/Articles/612021/>) as "relativistic hash tables" back in 2014. I thought at the time that it might be fun to make use of rhashtables, but didn't, until an opportunity arose through my work on the Lustre filesystem. Lustre is a cluster filesystem that is currently in `drivers/staging` while the code is revised to meet upstream requirements. One of those requirements is to avoid duplicating similar functionality where possible. As Lustre contains a resizable hash table, it really needs to be converted to use rhashtables instead — at last I have my opportunity. 

It didn't take me long to discover that the rhashtable implementation in Linux 4.15 is quite different from the one that originally landed in Linux 3.17, so the original LWN introduction is now barely relevant. I also quickly discovered that the in-kernel documentation was partially wrong, far from complete, and didn't provide any sort of "getting started" guide. Nevertheless I persisted and eventually developed a fairly complete understanding of the code, which seems worth sharing. This article gives an introduction to the use of the rhashtable interfaces without getting into too many internal implementation details. A followup will explain how rhashtables work internally and show how some of the mechanism details leak though the interfaces. 

#### Creating your first hash table

Suppose you have some data structure that you want to store in a hash table. This data structure will have a key and, usually, some other content. To work with rhashtables, the data structure must contain a `struct rhash_head` field which is used for table linkage, so the whole structure might be: 

```
struct object {
    	int key;
    	struct rhash_head linkage;
    	char content[64];
    	refcount_t ref;
    	struct rcu_head rcu_read;
        };
```

This example includes a reference counter and an `rcu_head` for lifetime management, which will be used in later examples. 

Creating a hash table to store some of these structures requires declaring some parameters and calling `rhashtable_init()`, for example: 

```
const static struct rhashtable_params object_params = {
    	.key_len     = sizeof(int),
    	.key_offset  = offsetof(struct object, key),
    	.head_offset = offsetof(struct object, linkage),
        };
    
        struct rhashtable my_objects;
    
        success = rhashtable_init(&my_objects, &object_params);
```

`rhashtable_init()` can fail with `-EINVAL` if some of the parameters given are invalid (this example only shows a small selection of the available parameters) or `-ENOMEM` if it failed to allocate some required memory. 

Providing just the length and offset of the key is sufficient for simple keys; rhashtables will provide code to compute a hash over the given number of bytes and do a byte-by-byte comparison to see if a given key matches an object. Often, keys are not that simple. Examples where length-plus-offset is not sufficient include: 

  * a variable-length key, such as a NUL-terminated string
  * a key that is not directly contained in the object, but is accessed through a pointer in the object
  * a key that is a C `struct` that might contain padding bytes. It is hard to guarantee these bytes will be zero, so a direct comparison is unlikely to work.

In such cases, a hash function (`.hashfn()`) and object comparison function (`.obj_cmpfn()`) can be provided with signatures matching: 

```
struct rhashtable_compare_arg {
    	struct rhashtable *ht;
    	const void *key;
        };
        typedef u32 (*rht_hashfn_t)(const void *data, u32 len, u32 seed);
        typedef int (*rht_obj_cmpfn_t)(struct rhashtable_compare_arg *arg,
    	         const void *obj);
```

When these are provided, the `key_len` is largely ignored, though it should be given a non-zero value since a `key_len` of zero indicates a more complex approach to hashing that is described later. The `obj_cmpfn()` should return zero for a match and non-zero otherwise, much like `memcmp()`. I like to return a negative error code, such as `-ESRCH`, rather than `1` as some in-kernel users do; it makes it more obvious when reading the code that zero means "success". 

#### Putting objects in the table

The most general way to insert into a hash table is to call: 

```
old_obj = rhashtable_lookup_get_insert_fast(&my_objects, &new_obj.linkage, object_params);
```

If an object already exists in the table with the same key, it is returned as `old_obj` and `new_obj` is not inserted. If no such old object exists, `new_obj` will normally be inserted and `NULL` will be returned. It is technically possible for insertion to fail, in which case an `ERR_PTR()` is returned. These error cases will be explained in the next article once the required context has been presented. 

If you don't want the old object (i.e. don't want to "get" it), or don't even want to check if it exists (no "lookup" wanted), other interfaces are available. The three main insert interfaces are: 

```
void *rhashtable_lookup_get_insert_fast(struct rhashtable *ht, struct rhash_head *obj,
    		const struct rhashtable_params params);
    
        int rhashtable_lookup_insert_fast(struct rhashtable *ht, struct rhash_head *obj,
    		const struct rhashtable_params params);
    
        int rhashtable_insert_fast(struct rhashtable *ht, struct rhash_head *obj,
    		const struct rhashtable_params params);
```

The fact that they all end in the word `fast` is probably historical accident; there are no "slow" versions. All rhashtable interfaces are "fast" in that they never sleep and can be called from interrupt handlers, while holding spinlocks, or in any other atomic context. Note that rhashtable_insert_fast() can insert duplicates in the table, so it must be used with caution. 

The passing of the `params` argument to these functions by value deserves some explanation. The parameters were already passed when the table was created, so the `ht` argument should already contain the parameter information — why pass it again? 

All of these insert functions, as well as the lookup and remove functions that we will meet shortly, are defined in [`include/linux/rhashtable.h`](<https://elixir.bootlin.com/linux/v4.16.1/source/include/linux/rhashtable.h>) as `static inline` functions, so the compiler generates a new copy of the code every place they are called. Because `params` is declared as a `const` by-value argument, the compiler can treat it as constant during code optimization and can discard code that is never used. In the example given above, no `hashfn()` is specified, so the code that ultimately gets run won't even test to see if there is a hash function. It will go directly to the default hash function (`jhash2()` in this case — the Jenkins hash function). This approach allows a lot of flexibility in the C code, with minimal overhead at run time. 

#### Lookup and object lifetimes

An object can be found in the hash table using `rhashtable_lookup_fast()`: 

```
void *rhashtable_lookup_fast(struct rhashtable *ht, const void *key,
    	const struct rhashtable_params params);
```

for example: 

```
int key = 42;
        struct object *found = rhashtable_lookup_fast(&my_objects, &key, object_params);
```

Note that there is a slight asymmetry between insert and lookup. When inserting an object, the address of the linkage (the `rhash_head` structure embedded in that object) is passed in. When looking up an object, the address of the whole object is returned. This is a stable pattern across the whole API and is easy to get used to: pass in the address of the linkage, get back the address of the object. 

When performing a lookup, you need always to be mindful of object lifetimes to ensure you don't find an object just as it is being deleted. In my previous experience with hash tables, each object in the table has a reference count which is incremented as part of the lookup operation while a hash-table lock is still held. Object removal takes the same lock and ensures that a newly looked-up object isn't removed. The resizable hash-table implementation in Lustre understands this and can be told how to increment or decrement a reference counter so that race-free lookup is easy. Rhashtables provide no such support. 

While this may seem like a weakness in rhashtables, it is really a strength. There are many different protocols that can be used to manage object lifetimes and, even with the fairly common reference-counter approach, there are different rules as to what happens when the count reaches zero. Encoding all these in the rhashtable code would be unwieldy; leaving it to the caller allows complete flexibility. 

One of the benefits of integrating object lifetimes with the hash table is that the hash-table lock can be leveraged to help with lifetimes. Without that benefit, a caller must provide their own locking. In many cases the RCU read lock is quite sufficient, and its cost is usually indistinguishable from zero. 

A common lifetime rule has an object removed from the hash table only when the reference count becomes zero. In this case there is a small window between zero being reached and the removal completing. This can be allowed for on lookup with: 

```
key = 42;
        rcu_read_lock();
        obj = rhashtable_lookup_fast(&my_objects, &key, params);
        if (obj && !refcount_inc_not_zero(&obj->ref))
    	obj = NULL;
        rcu_read_unlock();
```

#### Object removal

To remove an object from the table, use `rhashtable_remove_fast()`: 

```
int rhashtable_remove_fast(struct rhashtable *ht, struct rhash_head *obj,
    			       const struct rhashtable_params params);
```

This call will fail with `-ENOENT` if the object isn't found. There is no interface to remove an object given only the key, but you can easily combine lookup with removal: 

```
rcu_read_lock();
        obj = rhashtable_lookup_fast(table, key, params);
        if (obj && rhashtable_remove_fast(table, &obj->linkage, params) == 0)
    	kfree_rcu(obj, rcu_head);
        rcu_read_unlock();
```

When using RCU to help manage object lifetimes, it is important to use `kfree_rcu()` or similar to free the object, so the memory doesn't get reused while some other thread is performing a concurrent lookup, hence its inclusion in this example. 

To complete the example started earlier where an object is removed when the reference count becomes zero, the removal code might look like: 

```
if (refcount_dec_and_test(&obj->ref)) {
    	rhashtable_remove_fast(&my_objects, &obj->linkage, object_params);
    	kfree_rcu(obj, rcu_head);
        }
```

#### Walking all the objects in the table

It is occasionally useful to iterate (or "walk") over all the objects in a hash table. Rhashtables make this fairly easy, though there are some caveats. To support walking there is one data structure and six functions: 

```
struct rhashtable_iter;
    
        void rhashtable_walk_enter(struct rhashtable *ht, struct rhashtable_iter *iter);
        int rhashtable_walk_start_check(struct rhashtable_iter *iter);
        void rhashtable_walk_start(struct rhashtable_iter *iter);
        void *rhashtable_walk_next(struct rhashtable_iter *iter);
        void rhashtable_walk_stop(struct rhashtable_iter *iter);
        void rhashtable_walk_exit(struct rhashtable_iter *iter);
```

There are two different protocols for keeping track of the "current" location in the table during a walk, one that assumes the RCU read lock is held and one that does not. These can be mixed, so RCU might be held while walking for a while, then it might be dropped for a while, at which point the non-RCU protocol must be used to continue. This subtlety explains why there is both an enter/exit pair of functions, and a start/stop pair. The enter/exit pair must be called at the beginning and end, respectively, and they set up, then tear down, the iterator (`struct rhashtable_iter`). During a walk, the start/stop pair must be called at least once and all calls to `rhashtable_walk_next()` must occur between `rhashtable_walk_start()` and `rhashtable_walk_stop()`. 

To make this more concrete, a simple table walk that proceeds entirely under the RCU read lock might be: 

```
struct rhashtable_iter iter;
    
        rhashtable_walk_enter(&my_objects, &iter);
        rhashtable_walk_start(&iter);
        while ((obj = rhashtable_walk_next(&iter)) != NULL) {
    	if (IS_ERR(obj))
    	    continue;
    	printk_obj(obj);
        }
        rhashtable_walk_stop(&iter);
        rhashtable_walk_exit(&iter);
```

The RCU read lock is held between `rhashtable_walk_start()` and `rhashtable_walk_stop()` so nothing in the body of the walk is permitted to sleep. This walk is completely safe against concurrent insertions and removal: any object that was in the table at the start and is still there at the end will be printed. Objects inserted or removed during the walk may or may not be seen. It is quite safe to remove the object that you are visiting when walking like this. 

If the hash table is resized during the walk, it is possible to see duplicates and `rhashtable_walk_next()` can return with the error `-EAGAIN` at some point, hence the test. The implication of this will be detailed in the followup article. 

If the action on some objects needs to potentially sleep, then the walk must be temporarily "stopped", for example: 

```
struct rhashtable_iter iter;
    
        rhashtable_walk_enter(&my_objects, &iter);
        rhashtable_walk_start(&iter);
        while ((obj = rhashtable_walk_next(&iter)) != NULL) {
    	if (IS_ERR(obj))
    	    continue;
    	if (uninteresting(obj))
    	    continue;
    	if (!refcount_inc_not_zero(&obj->ref))
    	    continue;
    	rhashtable_walk_stop(&iter);
    	do_something_slow(obj);
    	rhashtable_walk_start(&iter);
    	obj_put(obj);
        }
        rhashtable_walk_stop(&iter);
        rhashtable_walk_exit(&iter);
```

When `rhashtable_walk_stop()` drops the RCU read lock, the rhashtable code can no longer be sure that `obj` will remain in the table, so `rhashtable_walk_next()` cannot just get the "next" object from the linkage structure. Instead it remembers which hash chain the object was in, and how many elements along the chain it was. If a concurrent insertion or removal changes the chain, the skip-count can be wrong and so the walk might miss an object, or see a duplicate. 

In our `my_objects` running example, we need to claim a reference to an object when dropping the RCU lock; this will prevent the object from being removed from the table. In this case the object is, in fact, still in the table, so `rhashtable_walk_start()` should be able to find it and restart the walk exactly where it left off. A patch has been submitted to implement this and, once accepted, it should be possible to walk the hash table and be guaranteed not to miss anything. Guaranteeing that no duplicates will be seen is not possible if concurrent resizes are permitted. The implications of this will be examined more closely when we dig into resizing next time. 

#### The rest of the API

That covers most of the interesting parts of the API, but for completeness we should quickly cover the remainder. 

When you have finished with an rhashtable it must be destroyed, typically with `rhashtable_free_and_destroy()`: 

```
void rhashtable_free_and_destroy(struct rhashtable *ht,
    			 void (*free_fn)(void *ptr, void *arg),
    			 void *arg);
```

This will call `free_fn()` on every object in the table, then free all internally allocated data structures. If there is nothing in the table, or if cleanup should be avoided for some other reason, then the simpler `rhashtable_destroy()` can be used. 

```
void rhashtable_destroy(struct rhashtable *ht);
```

The `rhashtable_lookup_and_insert()` interfaces we looked at above contain an implicit assumption that the key embedded in an object is in exactly the same format as the key that would be passed to `rhashtable_lookup()`. This is not always the case, as seen in an example from the [Mellanox Ethernet driver](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/ethernet/mellanox/mlxsw/spectrum_router.c?h=v4.15#n2720>) that has a lookup key with a linked list of values, and an object that contains an array of these values. As the `obj_cmpfn()` expects a lookup key and an object, this circumstance can only be managed if the insert function is passed both a key and an object. Further, a table like this will need two different hash functions, one to hash a standalone key and one to hash the key that is embedded in an object. 

To use rhashtable with this sort of key, three changes to the normal approach are needed: 

  * `.key_len` must be set to 0 (or left unset). 
  * A new function `.obj_hashfn` must be provided in the parameters. It has the same signature as `hashfn()` but is passed the object rather than the key. 
  * Instead of the `lookup_insert` functions already listed, one of the `insert_key` functions must be used: 

```
static inline void *rhashtable_lookup_get_insert_key(struct rhashtable *ht,
         			const void *key, struct rhash_head *obj,
        			const struct rhashtable_params params);
        
            static inline int rhashtable_lookup_insert_key(struct rhashtable *ht, 
        			const void *key, struct rhash_head *obj,
        			const struct rhashtable_params params);
```

Sometimes you might want a hash table to potentially contain multiple objects for any given key. In that case you can use "rhltables" — rhashtables with lists of objects. The interfaces are much the same as for rhashtables except that the table is represented by `struct rhltable` and the function names start with `rhltable_` instead of `rhashtable_`: 

```
int rhltable_init(struct rhltable *hlt, const struct rhashtable_params *params);
        struct rhlist_head *rhltable_lookup(struct rhltable *hlt, const void *key,
    	const struct rhashtable_params params);
        int rhltable_insert(struct rhltable *hlt, struct rhlist_head *list,
    	const struct rhashtable_params params);
        int rhltable_insert_key(struct rhltable *hlt, const void *key, 
    	struct rhlist_head *list, const struct rhashtable_params params);
        int rhltable_remove(struct rhltable *hlt, struct rhlist_head *list,
    	const struct rhashtable_params params);
        void rhltable_walk_enter(struct rhltable *hlt, struct rhashtable_iter *iter);
        void rhltable_free_and_destroy(struct rhltable *hlt,
        	void (*free_fn)(void *ptr, void *arg), void *arg);
```

#### What's next?

This completes the "getting started" introduction, which has already helped improve code quality as I found, while writing this, that some of my Lustre conversion was wrong. It should be enough for others to successfully use rhashtables in their own code, but it is not yet complete. There are more configuration parameters, and various subtleties related to resizing that need to be understood before you can be sure that all cases are covered. This will be explored in [the second half of this series](<https://lwn.net/Articles/751974/>).
