---
title: "RCU-safe reference counting"
url: https://lwn.net/Articles/93617/
date: "July 14, 2004"
category: "Race conditions; Read-copy-update; Reference counting; cmpxchg; kref"
author: 
---

The "`kref`" mechanism is a simple structure for implementing reference-counted objects in the kernel; it was covered here [last March](<http://lwn.net/Articles/75920/>). At the core of a `kref` is an `atomic_t` counter which contains the number of outstanding references. When that counter goes to zero, the object is no longer used and can be freed. 

The `kref` functions are simple. Obtaining a reference is done with a call to `kref_get()`: 

```
struct kref *kref_get(struct kref *kref)
        {
    	WARN_ON(!atomic_read(&kref->refcount));
    	atomic_inc(&kref->refcount);
    	return kref;
        }
```

Releasing that reference is accomplished with `kref_put()`: 

```
void kref_put(struct kref *kref)
        {
    	if (atomic_dec_and_test(&kref->refcount)) {
    	    kref->release(kref);
    	}
        }
```

The use of atomic types makes these functions safe in multiprocessor or preemptive environments; the reference count will always be correct. Except, of course, when things go wrong. Consider the following order of operations performed by two kernel threads; they could be running on separate processors, or on a preemptive, uniprocessor system: 

Thread 1| Thread 2  
---|---

```
/* In kref_get() */
    WARN_ON(!atomic_read(&kref->refcount));
```
  
  
|   
| 

```
kref_put(&kref);
```

```
atomic_inc(&kref->refcount);
    return kref;
```
  
  
|   
  
The first thread will be left thinking it holds a reference to an object which, in fact, has been deleted. As a general rule, good things cannot be expected to result from this situation. The `kref` code deals with this possibility by fiat: simultaneous calls to `kref_get()` and `kref_put()` on the same object are not allowed. In practice, this restriction usually requires that these operations be called under the protection of a lock somewhere. 

Developers interested in high-end scalability, however, often try to use lock-free algorithms. Locks can easily become a performance bottleneck as the number of threads increases, so, if they can be eliminated, the kernel will scale better. That is the motivation behind the use of techniques like [seqlocks](<https://lwn.net/Articles/22818/>) and [read-copy-update (RCU)](<https://lwn.net/Articles/37889/>). The locking requirement associated with the `kref` type makes that type difficult to use with these techniques. 

Ravikiran G Thirumalai recently posted a patch entitled "[Refcounting of objects part of a lockfree collection](<https://lwn.net/Articles/93565/>)" which implements a new locking type (called `refcount_t`) for dealing with objects managed using no-lock techniques. The explanation goes to great lengths to describe reference counting issued when working with RCU, but, in the end, all the patch is really doing, via a long path, is making a type which is like the `kref`, but which is not subject to the race described above. 

`kref_get()`, as currently written, checks the reference count first; if that count is zero, the object has already been freed. The current implementation merely complains when this happens; one could argue that stronger action is called for. The real problem, though, is that this test and the subsequent incrementing of the reference count are not, together, atomic - other actions can come between the two. Ravikiran's patch addresses this issue by coding his `_get()` function differently: 

```
static inline int refcount_get_rcu(refcount_t *rc)
        {
    	int c, old;
    	c = atomic_read(&rc->count);
    	while ( c && (old = cmpxchg(&rc->count.counter, c, c+1)) != c) 
    		c = old;
    	return c;
        }
```

The core of this function is the call to `cmpxchg()`, which is an inline assembly function giving access to the processor's `cmpxchg` instruction. The function prototype looks like: 

```
int cmpxchg(int *location, int old, int new);
```

(The actual definition is a little more complex, depending on the real type of `location`). The purpose of this function is to (1) compare the contents of `*location` with `old`, (2) if and only if the two are the same, assign `new` to `*location`, and (3) return the old value. If `cmpxchg()` returns `old`, the operation succeeded; otherwise the value pointed to by `location` is unchanged. The key point is that all of these operations are performed in an atomic manner 

`cmpxchg()` is, in other words, a form of test-and-set instruction. It is used here to increment the reference count in an atomic manner while being absolutely sure that nobody else can possibly have seen that count reach zero. When references are obtained in this way, the race described above cannot happen. 

There is still a pitfall, however. If the reference-counted object were to be freed and reused before another thread tried to obtain a reference, that thread might see a random "reference count" and think it succeeded. Preventing that turn of events is where RCU comes in. The actual object is freed by way of an RCU callback, which cannot happen until every processor has scheduled. If any thread can see a pointer to the object, said object will continue to exist, though its reference count may be zero. After a complete quiescence cycle, no threads can see such a pointer, and the object can be safely deleted. 

One other potential problem is that not all architectures offer a `cmpxchg` instruction. On such systems Ravikiran uses a rather more elaborate and unsightly scheme involving a hashed array of spinlocks; see the patch if morbid curiosity gets the better of you. 

This effort seems worthwhile; when this technique is used for [looking up file descriptors](<http://lwn.net/Articles/93566/>), tiobench performance improvements of 13% to 21% are claimed. There were objections, however, to the creation of a new reference counting API which is very similar to the `kref` API. As a result, the patch is likely to be rewritten to use `kref`s, extending that API as need be to supply the required semantics.
