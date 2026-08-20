---
title: A generic hash table
url: https://lwn.net/Articles/510202/
date: "August 8, 2012"
category: Hash table
author: "By Jake Edge August 8, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
August 8, 2012

A data structure implementation that is more or less replicated in 50 or more places in the kernel seems like some nice low-hanging fruit to pick. That is just what Sasha Levin is trying to do with his [generic hash table patch set](<https://lwn.net/Articles/510270/>). It implements a simple fixed-size hash table and starts the process of changing various existing hash table implementations to use this new infrastructure. 

The interface to Levin's hash table is fairly straightforward. The API is [defined](<https://lwn.net/Articles/510271/>) in `linux/hashtable.h` and one declares a hash table as follows: 

```
DEFINE_HASHTABLE(name, bits)
```

This creates a table with the given `name` and a power-of-2 size based on `bits`. The table is implemented using buckets containing a kernel `struct hlist_head` type. It implements a chaining hash, where hash collisions are simply added to the head of the `hlist`. One then calls: 

```
hash_init(name, bits);
```

to initialize the buckets. 

Once that's done, a structure containing a `struct hlist_node` pointer can be constructed to hold the data to be inserted, which is done with: 

```
hash_add(name, bits, node, key);
```

where `node` is a pointer to the `hlist_node` and `key` is the key that is hashed into the table. There are also two mechanisms to iterate over the table. The first iterates through the entire hash table, returning the entries in each bucket: 

```
hash_for_each(name, bits, bkt, node, obj, member)
```

The second returns only the entries that correspond to the `key`'s hash bucket: 

```
hash_for_each_possible(name, obj, bits, node, member, key)
```

In each case, `obj` is the type of the underlying data, `node` is a `struct hlist_head` pointer to use as a loop cursor, and `member` is the name of the `struct hlist_node` member in the stored data type. In addition, `hash_for_each()` needs an integer loop cursor, `bkt`. Beyond that, one can remove an entry from the table with: 

```
hash_del(node);
```

Levin has also converted six different hash table uses in the kernel as examples in the patch set. While the code savings aren't huge (a net loss of 16 lines), they could be reasonably significant after converting the 50+ different fixed-size hash tables that Levin [found](<https://lwn.net/Articles/510277/>) in the kernel. There is also the obvious advantage of restricting all of the hash table implementation bugs to one place. 

There has been a fair amount of discussion of the patches over the three revisions that Levin has posted so far. Much of it concerned implementation details, but there was another more global concern as well. Eric W. Biederman was [not convinced](<https://lwn.net/Articles/510279/>) that replacing the existing simple hash tables was desirable: 

For a trivial hash table I don't know if the abstraction is worth it. For a hash table that starts off small and grows as big as you need it the [incentive] to use a hash table abstraction seems a lot stronger. 

But, Linus Torvalds [disagreed](<https://lwn.net/Articles/510280/>). He mentioned that he had been ""playing around"" with a directory cache (dcache) patch that uses a fixed-size hash table as an L1 cache for directory entries that provided a noticeable performance boost. If a lookup in that first hash table fails, the code then falls back to the existing dynamically sized hash table. The reason that the code hasn't been committed yet is because ""filling of the small L1 hash is racy for me right now"" and he has not yet found a lockless and race-free way to do so. So: 

[...] what I really wanted to bring up was the fact that static hash tables of a fixed size are really quite noticeably faster. So I would say that Sasha's patch to make *that* case easy actually sounds nice, rather than making some more complicated case that is fundamentally slower and more complicated. 

Torvalds [posted his patch](<https://lwn.net/Articles/510283/>) (dropped [diff attachment](<https://lwn.net/Articles/510313/>)) after a request from Josh Triplett. The race condition is ""almost entirely theoretical"", he said, so it could be used to generate some preliminary performance numbers. Beyond just using the small fixed-sized table, Torvalds's patch also circumvents any chaining; if the hash bucket doesn't contain the entry, the second cache is consulted. By [avoiding "pointer chasing"](<https://lwn.net/Articles/509417/>), the L1 dcache ""really improved performance"". 

Torvalds's dcache work is, of course, something of an aside in terms of Levin's patches, but several kernel developers seemed favorably inclined toward consolidating the various kernel hash table implementations. Biederman was [unimpressed](<https://lwn.net/Articles/510287/>) with the conversion of the UID cache in the user namespace code and Nacked it. On the other hand, Mathieu Desnoyers had [only minor comments](<https://lwn.net/Articles/510289/>) on the conversion of the tracepoint hash table and Eric Dumazet had mostly [stylistic comments](<https://lwn.net/Articles/510290/>) on the conversion of the 9p protocol error table. There are several other maintainers who have not yet weighed in, but so far most of the reaction has been positive. Levin is trying to attract more reviews by converting a few subsystems, as he notes in the patch. 

It is still a fair amount of work to convert the other 40+ implementations, but the conversion seems fairly straightforward. But, Biederman's complaint about the conversion of the namespace code is something to note: ""I don't have the time for a new improved better hash table that makes the code buggier."" Levin will need to prove that his implementation works well, and that the conversions don't introduce regressions, before there is any chance that we will see it in the mainline. There is no reason that all hash tables need to be converted before that happens--though it might make it more likely to go in.
