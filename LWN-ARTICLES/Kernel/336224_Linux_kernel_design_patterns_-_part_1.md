---
title: "Linux kernel design patterns - part 1"
url: https://lwn.net/Articles/336224/
date: "June 8, 2009"
category: "Development model-Patterns"
author: "June 8, 2009 This article was contributed by Neil Brown"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

June 8, 2009

This article was contributed by Neil Brown

One of the topics of ongoing interest in the kernel community is that of maintaining quality. It is trivially obvious that we need to maintain and even improve quality. It is less obvious how best to do so. One broad approach that has found some real success is to increase the visibility of various aspects of the kernel. This makes the quality of those aspects more apparent, so this tends to lead to an improvement of the quality. 

This increase in visibility takes many forms: 

  * The `checkpatch.pl` script highlights many deviations from accepted code formatting style. This encourages people (who remember to use the script) to fix those style errors. So, by increasing the visibility of the style guide, we increase the uniformity of appearance and so, to some extent, the quality. 

  * The "lockdep" system (when enabled) dynamically measures dependencies between locks (and related states such as whether interrupts are enabled). It then reports anything that looks odd. These oddities will not always mean a deadlock or similar problem is possible, but in many cases they do, and the deadlock possibility can be removed. So by increasing the visibility of the lock dependency graph, quality can be increased. 

  * The kernel contains various other enhancements to visibility such as the "poisoning" of unused areas of memory so invalid access will be more apparent, or simply the use of symbolic names rather than plain hexadecimal addresses in stack traces so that bug reports are more useful. 

  * At a much higher level, the "git" revision tracking software that is used for tracking kernel changes makes it quite easy to see who did what and when. The fact that it encourages a comment to be attached to each patch makes it that much easier to answer the question "Why is the code this way". This visibility can improve understanding and that is likely to improve quality as more developers are better informed. 

There are plenty of other areas where increasing visibility does, or could, improve quality. In this series we will explore one particular area where your author feels visibility could be increased in a way that could well lead to qualitative improvements. That area is the enunciation of kernel-specific design patterns. 

#### Design Patterns

A "design pattern" is a concept that was first expounded in the field of Architecture and was brought to computer engineering, and particularly the Object Oriented Programming field, though the 1994 publication _Design Patterns: Elements of Reusable Object-Oriented Software_. Wikipedia has [further useful background information](<http://en.wikipedia.org/wiki/Design_pattern_%28computer_science%29>) on the topic. 

In brief, a design pattern describes a particular class of design problem, and details an approach to solving that class of problem that has proven effective in the past. One particular benefit of a design pattern is that it combines the problem description and the solution description together and gives them a name. Having a simple and memorable name for a pattern is particularly valuable. If both developer and reviewer know the same names for the same patterns, then a significant design decision can be communicated in one or two words, thus making the decision much more visible. 

In the Linux kernel code base there are many design patterns that have been found to be effective. However most of them have never been documented so they are not easily available to other developers. It is my hope that by explicitly documenting these patterns, I can help them to be more widely used and, thus, developers will be able to achieve effective solutions to common problems more quickly. 

In the remainder of this series we will be looking at three problem domains and finding a variety of design patterns of greatly varying scope and significance. Our goal in doing so is to not only to enunciate these patterns, but also to show the range and value of such patterns so that others might make the effort to enunciate patterns that they have seen. 

A number of examples from the Linux kernel will be presented throughout this series as examples are an important part of illuminating any pattern. Unless otherwise stated they are all from 2.6.30-rc4. 

#### Reference Counts

The idea of using a reference counter to manage the lifetime of an object is fairly common. The core idea is to have a counter which is incremented whenever a new reference is taken and decremented when a reference is released. When this counter reaches zero any resources used by the object (such as the memory used to store it) can be freed. 

The mechanisms for managing reference counts seem quite straightforward. However there are some subtleties that make it quite easy to get the mechanisms wrong. Partly for this reason, the Linux kernel has (since 2004) a data type known as "kref" with associated support routines (see `Documentation/kref.txt`, `<linux/kref.h>`, and `lib/kref.c`). These encapsulate some of those subtleties and, in particular, make it clear that a given counter is being used as a reference counter in a particular way. As noted above, names for design patterns are very valuable and just providing that name for kernel developers to use is a significant benefit for reviewers. 

In [the words of Andrew Morton](<http://lkml.indiana.edu/hypermail/linux/kernel/0403.1/1656.html>): 

I care more about being able to say "ah, it uses kref. I understand that refcounting idiom, I know it's well debugged and I know that it traps common errors". That's better than "oh crap, this thing implements its own refcounting - I need to review it for the usual errors". 

This inclusion of kref in the Linux kernel gives both a tick and a cross to the kernel in terms of explicit support for design patterns. A tick is deserved as the kref clearly embodies an important design pattern, is well documented, and is clearly visible in the code when used. It deserves a cross however because the kref only encapsulates part of the story about reference counting. There are some usages of reference counting that do not fit well into the kref model as we will see shortly. Having a "blessed" mechanism for reference counting that does not provide the required functionality can actually encourage mistakes as people might use a kref where it doesn't belong and so think it should just work where in fact it doesn't. 

A useful step to understanding the complexities of reference counting is to understand that there are often two distinctly different sorts of references to an object. In truth there can be three or even more, but that is very uncommon and can usually be understood by generalizing the case of two. We will call the two types of references "external" and "internal", though in some cases "strong" and "weak" might be more appropriate. 

An "external" reference is the sort of reference we are probably most accustomed to think about. They are counted with "get" and "put" and can be held by subsystems quite distant from the subsystem that manages the object. The existence of a counted external reference has a strong and simple meaning: This object is in use. 

By contrast, an "internal" reference is often not counted, and is only held internally to the system that manages the object (or some close relative). Different internal references can have very different meanings and hence very different implications for implementation. 

Possibly the most common example of an internal reference is a cache which provides a "lookup by name" service. If you know the name of an object, you can apply to the cache to get an external reference, providing the object actually exists in the cache. Such a cache would hold each object on a list, or on one of a number of lists under e.g. a hash table. The presence of the object on such a list is a reference to the object. However it is likely not a counted reference. It does not mean "this object is in use" but only "this object is hanging around in case someone wants it". Objects are not removed from the list until all external references have been dropped, and possibly they won't be removed immediately even then. Clearly the existence and nature of internal references can have significant implications on how reference counting is implemented. 

One useful way to classify different reference counting styles is by the required implementation of the "put" operation. The "get" operation is always the same. It takes an external reference and produces another external reference. It is implemented by something like: 

```
assert(obj->refcount > 0) ; increment(obj->refcount);
```

or, in Linux-kernel C: 

```
BUG_ON(!atomic_read(&obj->refcnt)) ; atomic_inc(&obj->refcnt);
```

Note that "get" cannot be used on an unreferenced object. Something else is needed there. 

The "put" operation comes in three variations. While there can be some overlap in use cases, it is good to keep them separate to help with clarity of the code. The three options, in Linux-C, are: 

```
1      atomic_dec(&obj->refcnt);
    
       2      if (atomic_dec_and_test(&obj->refcnt)) { ... do stuff ... }
    
       3      if (atomic_dec_and_lock(&obj->refcnt, &subsystem_lock)) {
                     ..... do stuff ....
    		 spin_unlock(&subsystem_lock);
    	  }
```

#### The "kref" style

Starting in the middle, option "2" is the style used for a kref. This style is appropriate when the object does not outlive its last external reference. When that reference count becomes zero, the object needs to be freed or otherwise dealt with, hence the need to test for the zero condition with `atomic_dec_and_test()`. 

Objects that fit this style often do not have any internal references to worry about, as is the case with most objects in sysfs, which is a heavy user of kref. If, instead, an object using the kref style does have internal references, it cannot be allowed to create an external reference from an internal reference unless there are known to be other external references. If this is necessary, a primitive is available: 

```
atomic_inc_not_zero(&obj->refcnt);
```

which increments a value providing it isn't zero, and returns a result indicating success or otherwise. `atomic_inc_not_zero()` is a relatively recent invention in the linux kernel, appearing in late 2005 as part of the lockless page cache work. For this reason it isn't widely used and some code that could benefit from it uses spinlocks instead. Sadly, the kref package does not make use of this primitive either. 

An interesting example of this style of reference that does not use kref, and does not even use `atomic_dec_and_test()` (though it could and arguably should) are the two ref counts in `struct super`: `s_count` and `s_active`. 

`s_active` fits the kref style of reference counts exactly. A superblock starts life with `s_active` being 1 (set in `alloc_super()`), and, when `s_active` becomes zero, further external references cannot be taken. This rule is encoded in `grab_super()`, though this is not immediately clear. The current code (for historical reasons) adds a very large value (S_BIAS) to `s_count` whenever `s_active` is non-zero, and `grab_super()` tests for `s_count` exceeding S_BIAS rather than for `s_active` being zero. It could just as easily do the latter test using `atomic_inc_not_zero()`, and avoid the use of spinlocks. 

`s_count` provides for a different sort of reference which has both "internal" and "external" aspects. It is internal in that its semantic is much weaker than that of `s_active`-counted references. References counted by `s_count` just mean "this superblock cannot be freed just now" without asserting that it is actually active. It is external in that it is much like a kref starting life at 1 (well, 1*S_BIAS actually), and when it becomes zero (in `__put_super()`) the superblock is destroyed. 

So these two reference counts could be replaced by two krefs, providing: 

  * S_BIAS was set to 1 

  * `grab_super()` used `atomic_inc_not_zero()` rather than testing against S_BIAS 

and a number of spinlock calls could go away. The details are left as an exercise for the reader. 

#### The "kcref" style

The Linux kernel doesn't have a "kcref" object, but that is a name that seems suitable to propose for the next style of reference count. The "c" stands for "cached" as this style is very often used in caches. So it is a Kernel Cached REFerence. 

A kcref uses `atomic_dec_and_lock()` as given in option 3 above. It does this because, on the last put, it needs to be freed or checked to see if any other special handling is needed. This needs to be done under a lock to ensure no new reference is taken while the current state is being evaluated. 

A simple example here is the `i_count` reference counter in `struct inode`. The important part of `iput()` reads: 

```
if (atomic_dec_and_lock(&inode->i_count, &inode_lock))
    	iput_final(inode);
```

where `iput_final()` examines the state of the inode and decides if it can be destroyed, or left in the cache in case it could get reused soon. 

Among other things, the `inode_lock` prevents new external references being created from the internal references of the inode hash table. For this reason converting internal references to external references is only permitted while the `inode_lock` is held. It is no accident that the function supporting this is called `iget_locked()` (or `iget5_locked()`). 

A slightly more complex example is in `struct dentry`, where `d_count` is managed like a kcref. It is more complex because two locks need to be taken before we can be sure no new reference can be taken - both `dcache_lock` and `de->d_lock`. This requires that either we hold one lock, then `atomic_dec_and_lock()` the other (as in `prune_one_dentry()`), or that we `atomic_dec_and_lock()` the first, then claim the second and retest the refcount - as in `dput()`. This is good example of the fact that you can never assume you have encapsulated all possible reference counting styles. Needing two locks could hardly be foreseen. 

An even more complex kcref-style refcount is `mnt_count` in `struct vfsmount`. The complexity here is the interplay of the two refcounts that this structure has: `mnt_count`, which is a fairly straightforward count of external references, and `mnt_pinned`, which counts internal references from the process accounting module. In particular it counts the number of accounting files that are open on the filesystem (and as such could use a more meaningful name). The complexity comes from the fact that when there are only internal references remaining, they are all converted to external references. Exploring the details of this is again left as an exercise for the interested reader. 

#### The "plain" style

The final style for refcounting involves just decrementing the reference count (`atomic_dec()`) and not doing anything else. This style is relatively uncommon in the kernel, and for good reason. Leaving unreferenced objects just lying around isn't a good idea. 

One use of this style is in `struct buffer_head`, managed by `fs/buffer.c` and `<linux/buffer_head.h>`. The `put_bh()` function is simply: 

```
static inline void put_bh(struct buffer_head *bh)
        {
            smp_mb__before_atomic_dec();
            atomic_dec(&bh->b_count);
        }
```

This is OK because buffer_heads have lifetime rules that are closely tied to a page. One or more buffer_heads get allocated to a page to chop it up into smaller pieces (buffers). They tend to remain there until the page is freed at which point all the buffer_heads will be purged (by `drop_buffers()` called from `try_to_free_buffers()`). 

In general, the "plain" style is suitable if it is known that there will always be an internal reference so that the object doesn't get lost, and if there is some process whereby this internal reference will eventually get used to find and free the object. 

#### Anti-patterns

To wrap up this little review of reference counting as an introduction to design patterns, we will discuss the related concept of an anti-pattern. While design patterns are approaches that have been shown to work and should be encouraged, anti-patterns are approaches that history shows us do not work well and should be discouraged. 

Your author would like to suggest that the use of a "bias" in a refcount is an example of an anti-pattern. A bias in this context is a large value that is added to, or subtracted from, the reference count and is used to effectively store one bit of information. We have already glimpsed the idea of a bias in the management of `s_count` for superblocks. In this case the presence of the bias indicates that the value of `s_active` is non-zero, which is easy enough to test directly. So the bias adds no value here and only obscures the true purpose of the code. 

Another example of a bias is in the management of `struct sysfs_dirent`, in `fs/sysfs/sysfs.h` and `fs/sysfs/dir.c`. Interestingly, `sysfs_dirent` has two refcounts just like superblocks, also called `s_count` and `s_active`. In this case `s_active` has a large negative bias when the entry is being deactivated. The same bit of information could be stored just as effectively and much more clearly in the flag word `s_flags`. Storing single bits of information in flags is much easier to understand that storing them as a bias in a counter, and should be preferred. 

In general, using a bias does not add any clarity as it is not a common pattern. It cannot add more functionality than a single flag bit can provide, and it would be extremely rare that memory is so tight that one bit cannot be found to record whatever would otherwise be denoted by the presence of the bias. For these reasons, biases in refcounts should be considered anti-patterns and avoided if at all possible. 

#### Conclusion

This brings to a close our exploration of the various design patterns surrounding reference counts. Simply having terminology such a "kref" versus "kcref" and "external" versus "internal" references can be very helpful in increasing the visibility of the behaviour of different references and counts. Having code to embody this as we do with kref and could with kcref, and using this code at every opportunity, would be a great help both to developers who might find it easy to choose the right model first time, and to reviewers who can see more clearly what is intended. 

The design patterns we have covered in this article are: 

  * **kref** : When the lifetime of an object extends only to the moment that the last external reference is dropped, a kref is appropriate. If there are any internal reference to the object, they can only be promoted to external references with `atomic_inc_not_zero()`. Examples: `s_active` and `s_count` in `struct super_block`. 

  * **kcref** : With this the lifetime of an object can extend beyond the dropping of the last external reference, the kcref with its `atomic_dec_and_lock()` is appropriate. An internal reference can only be converted to an external reference will the subsystem lock is held. Examples: `i_count` in `struct inode`. 

  * **plain** : When the lifetime of an object is subordinate to some other object, the plain reference pattern is appropriate. Non-zero reference counts on the object must be treated as internal reference to the parent object, and converting internal references to external references must follow the same rules as for the parent object. Examples: `b_count` in `struct buffer_head`. 

  * **biased-reference** : When you feel the need to use add a large bias to the value in a reference count to indicate some particular state, don't. Use a flag bit elsewhere. This is an anti-pattern. 

Next week we will move on to another area where the Linux kernel has proved some successful design patterns and explore the slightly richer area of complex data structures. ([Part 2](<https://lwn.net/Articles/336255/>) and [part 3](<https://lwn.net/Articles/336262/>) of this series are now available). 

#### Exercises

As your author has been reminded while preparing this series, there is nothing like a directed study of code to clarify understanding of these sorts of issues. With that in mind, here are some exercises for the interested reader. 

  1. Replace `s_active` and `s_count` in `struct super` with krefs, discarding S_BIAS in the process. Compare the result with the original using the trifecta of Correctness, Maintainability, and Performance. 

  2. Choose a more meaningful name for `mnt_pinned` and related functions that manipulate it. 

  3. Add a function to the kref library that makes use of `atomic_inc_not_zero()`, and using it (or otherwise) remove the use of `atomic_dec_and_lock()` on a kref in `net/sunrpc/svcauth.c` \- a usage which violates the kref abstraction. 

  4. Examine the `_count` reference count in `struct page` (see `mm_types.h` for example) and determine whether it behaves most like a kref or a kcref (hint: it is not "plain"). This should involve identifying any and all internal references and related locking rules. Identify why the page cache (`struct address_space.page_tree`) owns a counted reference or explain why it should not. This will involve understanding `page_freeze_refs()` and its usage in `__remove_mapping()`, as well as `page_cache_{get,add}_speculative()`. 

Bonus credit: provide a series of minimal self-contained patches to implement any changes that the above investigations proved useful.
