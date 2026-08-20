---
title: "Relativistic hash tables, part 2: Implementation"
url: https://lwn.net/Articles/612100/
date: "September 17, 2014"
category: "Hash table; Read-copy-update"
author: "By Jonathan Corbet September 17, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 17, 2014

The kernel's "relativistic" (or "resizable") hash table implementation came into the mainline during the 3.17 merge window. These tables can be resized while still allowing concurrent access by readers and (to an extent) updaters. See [the first article in this series](<https://lwn.net/Articles/612021/>) for a description of the algorithms used to implement this functionality. This article, instead, will describe the in-kernel API for these "rhashtables" as seen in 3.17, with an eye toward changes that are likely to arrive in the near future. 

Rhashtables were initially created for use in the networking subsystem, but their implementer, Thomas Graf, understood that they could be more widely used within the kernel. As a result, the API is fairly well generalized. One implication of this design is that a certain amount of setup work is needed to put an rhashtable into use. Once that setup is done, though, operations on the table are relatively straightforward. 

The first step is to define the object that will be stored in the hash table. Such objects will typically look something like this: 

```
#include <linux/rhashtable.h>
    
        struct hashed_object {
    	int key;
    	struct rhash_head node;
    	/* Stuff of actual interest */
        }
```

Here, an integer key is used for identifying objects in the table; the implementation can handle just about any key type, though, as will be seen below. 

The next step is to fill in a structure describing how the hash table will work. This structure is defined as: 

```
typedef u32 (*rht_hashfn_t)(const void *data, u32 len, u32 seed);
        typedef u32 (*rht_obj_hashfn_t)(const void *data, u32 seed);
    
        struct rhashtable_params {
    	size_t		  nelem_hint;
    	size_t		  key_len;
    	size_t		  key_offset;
    	size_t		  head_offset;
    	u32		  hash_rnd;
    	size_t		  max_shift;
    	rht_hashfn_t	  hashfn;
    	rht_obj_hashfn_t  obj_hashfn;
    	bool		  (*grow_decision)(const struct rhashtable *ht,
    					   size_t new_size);
    	bool		  (*shrink_decision)(const struct rhashtable *ht,
    					     size_t new_size);
    	int		  (*mutex_is_held)(void);
        };
```

Here, `nelem_hint` is a guess at how many elements will be stored in the table (it is used to calculate the initial size). `key_len` and `key_offset` describe the key — its size in bytes and its offset into the structure (which should be obtained with `offsetof()`). Similarly, `head_offset` is the offset of the `rhash_head` structure within the hashed object. `hash_rnd` is a random seed to be used in the hash function; if it is zero, the rhashtable code will generate a random seed. The maximum size of the table can be optionally set with `max_shift`; its value is the base-two logarithm of that size. `hashfn` is the function that performs the hashing; normally it can be set to `arch_fast_hash()`, defined in `<linux/hash.h>`. `obj_hashfn()`, instead, is a function to hash the entire object. If `key_len` and `key_offset` have been provided, there is no need for `obj_hashfn()`. If calculating the hash requires something more complicated than looking at a few contiguous bytes in the object, this function can be provided to do the work (and `key_len` should be set to zero). 

Control over automatic resizing is provided by the `grow_decision()` and `shrink_decision()` functions. Despite being called `new_size`, that parameter actually holds the current number of buckets in the table; the number of elements can be found in `ht->elems`. If these functions return a true value, the size of the table will be doubled or halved as appropriate. The rhashtable implementation offers two functions that can be used in this role: `rht_grow_above_75()` and `rht_shrink_below_30()`. These functions seek to keep between 30% and 75% of the table buckets full. 

Finally, `mutex_is_held()` must be provided; it returns true if the lock serializing changes to the table is currently held. It is used purely for debugging purposes, ensuring that the lock is held when functions that modify the table are called. 

Once that structure is set up, it's just a matter of allocating a `struct rhashtable` instance and initializing it with: 

```
int rhashtable_init(struct rhashtable *ht, 
    			struct rhashtable_params *params);
```

There is also an `rhashtable_destroy()` to free a hash table's resources. 

Insertion and removal are handled with: 

```
void rhashtable_insert(struct rhashtable *ht, struct rhash_head *node, 
    			   gfp_t gfp_flags);
        bool rhashtable_remove(struct rhashtable *ht, struct rhash_head *node, 
    			   gfp_t gfp_flags);
        void rhashtable_remove_pprev(struct rhashtable *ht, struct rhash_head *obj,
    			         struct rhash_head __rcu **pprev, 
    				 gfp_t gfp_flags);
```

The table lock must be held when any of the above functions are called. In all of these calls, `ht` refers to the hash table, `node` points to the `rhash_head` structure within the relevant object, and `gfp_flags` is used for memory allocation should the table need to be resized. If the caller happens to have a pointer to the object immediately prior to the one that is to be removed, it can speed up the removal process by calling `rhashtable_remove_pprev()`. 

Note that these functions are likely to change in the near future. A [patch set](<https://lwn.net/Articles/611951/>) currently in review moves resizing operations into a separate thread, so there is no longer a need to allocate memory in the above functions. Accordingly, the `gfp_flags` argument is removed. This change makes it easier to perform hash table operations while running in atomic context. 

Should there be a reason to explicitly change the size of an rhashtable, that can be done with: 

```
int rhashtable_expand(struct rhashtable *ht, gfp_t gfp_flags);
        int rhashtable_shrink(struct rhashtable *ht, gfp_t gfp_flags);
```

The table lock must be acquired before calling these functions. A call to `rhashtable_expand()` will double the size of the table, while `rhashtable_shrink()` will halve its size. Again, the `gfp_flags` argument is likely to be removed in 3.18 or shortly thereafter. 

Lookups are performed with either of: 

```
void *rhashtable_lookup(const struct rhashtable *ht, const void *key);
        void *rhashtable_lookup_compare(const struct rhashtable *ht, u32 hash,
    				    bool (*compare)(void *, void *), 
    				    void *arg);
```

The first will simply return the object in the table matching `key`, or `NULL` if no such object exists. In the second form, an explicit comparison function may be passed to determine whether a given object matches or not. In this case, `hash` is a precomputed hash for the desired object, usually obtained from `rhashtable_hashfn()`. The comparison function is `compare()`; its arguments are the object to be tested and `arg`. 

The latter function is clearly meant for code which delves more deeply into the structure of the hash tables themselves. For cases where that is necessary, the interface provides a whole set of functions for traversing the hash chains; see `<linux/rhashtable.h>` for the full list, and at `net/netlink/af_netlink.c` for an example of their use. 

Future changes call for changing the hash chain into a "nulls list," where the `NULL` pointer at the end of the list contains the hash of the first item in the bucket. Such lists are used heavily in the networking subsystem where things can change on the fly and an object may move between lists while it is being traversed. This change won't affect relatively casual users of rhashtables, but code that follows the hash chains itself will need to adapt. 

For those wanting more details on how this code works, it can be found in `lib/rhashtable.c`. The code is relatively clear, well documented, and easy to follow, especially in its 3.17 incarnation; the changes to come will necessarily add complexity.
