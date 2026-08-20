---
title: "Relativistic hash tables, part 1: Algorithms"
url: https://lwn.net/Articles/612021/
date: "September 17, 2014"
category: "Hash table; Read-copy-update"
author: "By Jonathan Corbet September 17, 2014"
---

By **Jonathan Corbet**  
September 17, 2014

[Hash tables](<http://en.wikipedia.org/wiki/Hash_table>) are heavily used within the kernel to speed access to objects of interest. Using a hash table will be faster than, say, a linear search through a single list, but there is always value in making accesses faster yet. Quite a bit of work has been done toward this goal over the years; for example, the use of read-copy-update (RCU) can allow the addition and removal of items from a hash bucket list without locking out readers. Some operations have proved harder to make concurrent in that manner, though, with table resizing being near the top of the list. As of 3.17, the Linux kernel has gotten around this problem with an implementation of "relativistic hash tables" that can be resized while lookups proceed concurrently. This article will describe the algorithm that is used; a [companion article](<https://lwn.net/Articles/612100/>) will look at the kernel API for these new hash tables. 

One might wonder whether the resizing of hash tables is common enough to be worth optimizing. As it turns out, picking the correct size for a hash table is not easy; the kernel has many tables whose size is determined at system initialization time with a combination of heuristics and simple guesswork. But even if the initial guess is perfect, workloads can vary over time. A table that was optimally sized may, after a change, end up too small (and thus perform badly) or too big (wasting memory). Resizing the table would fix these problems, but, since that is hard to do without blocking access to the table, it tends not to happen. The longer-term performance gains are just not seen to be worth the short-term latency caused by shutting down access to the table while it is resized. 

Back in 2010, Josh Triplett, Paul McKenney, and Jonathan Walpole published a paper called [Resizable, Scalable, Concurrent Hash Tables via Relativistic Programming [PDF]](<https://www.usenix.org/legacy/event/atc11/tech/final_files/Triplett.pdf>) that described a potential solution to this problem. "Relativistic" refers to the fact that the relative timing of two events (hash table insertions, say) that are not causally related may appear different to independent observers. In other words, one CPU may see two items inserted into the table in one order, while those insertions appear to have happened in the opposite order on another CPU. Despite some interesting performance results, this technique did not find its way into the kernel until the 3.17 merge window opened, when Thomas Graf contributed an implementation for use within the networking subsystem. 

For the most part, relativistic hash tables resemble the ordinary variety. They are implemented as an array of buckets and a hash function to assign objects to buckets. Each bucket contains a linked list of all items in the table that belong to that bucket. A simple table with two buckets might thus be portrayed something like this: 

> ![\[Hash table\]](https://static.lwn.net/images/2014/rht/hashtable.png)

Here, the first bucket has four items, the second bucket has three. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Table shrinking

Now imagine that this massive hash table is deemed to be too large; it would be nice to be able to shrink it down to a single bucket. (No need to point out that a single-bucket hash table is useless; the point is to simplify the example as much as possible). The first step is to allocate the new table (without making it visible to anybody else) and to link each bucket to the first list in the old table that would hash to that bucket in the new table: 

> ![\[Shrinking\]](https://static.lwn.net/images/2014/rht/shrink1.png)

Note the assumption that all items falling into the same bucket in the old table will also fall into the same bucket in the new table. That is a requirement for this algorithm to work. In practice, it means that the size of a table can only change by an integer factor; normally that means that the size of a table can only be doubled or halved. 

Anyway, the new table now has some of the necessary items (the green ones) in its sole bucket, but the blue ones are still not represented there. That can be fixed by simply splicing them on to the end of the green list: 

> ![\[Shrinking\]](https://static.lwn.net/images/2014/rht/shrink2.png)

At this point, a reader traversing the green list may wander into the blue items added to the end. Since a hash lookup must always compare the keys anyway, the result will still be correct, it will just take a little longer than it did before. But that is how things will work once the operation is complete anyway. Completing simply requires pointing to the new table rather than the old: 

> ![\[Shrinking\]](https://static.lwn.net/images/2014/rht/shrink3.png)

Any new readers that come along after this point will see the new table, but there could still be concurrent readers traversing the lists in the old table. So the resize implementation must wait for an RCU grace period to pass; after that the old table can be freed and the operation is complete. 

#### Table expansion

Making a table larger is a somewhat different operation, because objects appearing in a single hash bucket in the old table must be split among multiple buckets in the new table. Imagine that we start with the same two-bucket table: 

> ![\[Expanding\]](https://static.lwn.net/images/2014/rht/exp0.png)

We now wish to expand that table to four buckets. The colors of the objects in the diagram are meant to indicate which bucket each object will land in once the resizing is complete. The first step is to allocate the new table, and, for each bucket in the new table, set a pointer to the chain containing all of the objects that belong in that bucket: 

> ![\[Expanding\]](https://static.lwn.net/images/2014/rht/exp1.png)

Note that each pointer from the new table's hash bucket goes to the first item in the old chain that would hash to the _new_ bucket. So, while each hash chain in the new table contains objects that do not belong there, the first object in the list is always in the correct bucket. At this point, the new table can be used for lookups, though they will be slower than they need to be due to the need to pass over objects that appear to be in the wrong list. 

To make the new table visible to readers, the pointer to the hash table must be changed accordingly. The expansion code must then wait for an RCU grace period to pass so that it can be sure that there are no longer any lookups running from the old table. Then the process of cleaning up ("unzipping") the lists in the new table can begin. 

That process works by passing over the buckets in the _old_ table, which has not yet been freed. For each bucket, the algorithm is somewhat like this: 

  1. Determine which bucket (in the new table) the first item in the list belongs to. 

  2. Advance list head pointer through the list until it encounters the first item that does _not_ belong in the same bucket. 

The result after processing the first bucket would look something like: 

> ![\[Expanding\]](https://static.lwn.net/images/2014/rht/exp2.png)

Then, the mismatched object (and any like it that immediately follow) are patched out of the list by changing the "next" pointer in the previous item: 

> ![\[Expanding\]](https://static.lwn.net/images/2014/rht/exp3.png)

At this point, no other changes can be made to this list until another RCU grace period has passed, otherwise a reader following the lists could end up in the wrong place. Remember that readers can be following these lists concurrently with the above-described change, but they will be starting from the pointers in the new table. A reader looking for a light green object may be looking at the dark green object that is patched out of the list in the first step, but, since that object's "next" pointer remains unchanged, the reader will not be derailed. Readers looking for dark green objects will have never seen the changed object to begin with, so they, too, are safe. 

In other words, the algorithm works by changing pointers that are visible only to readers that are not looking for the objects that will be "unzipped" from the list. As long as only one pointer in the list is changed for each RCU grace period, this property holds and the list can be safely unzipped while readers pass through it. Note that each bucket (from the old table) can be processed in this way in each cycle, so the second bucket in this example can also be partially unzipped. So the data structure will look like this when the wait for the grace period happens after the first pass through the old table: 

> ![\[Expanding\]](https://static.lwn.net/images/2014/rht/exp4.png)

The hash lists are now partially straightened out, but the job is not yet complete. So the unzipping process runs again, tweaking another "next" pointer in each list if needed. This process continues until it reaches the end of each of the hash lists, at which point each object appears only in its appropriate bucket: 

> ![\[Finished\]](https://static.lwn.net/images/2014/rht/done.png)

If the hash lists are long, this process may take a fair amount of time. But resize operations should be relatively rare, while readers access the table frequently. So, for a sufficiently read-mostly table, the extra effort will be worth it in the end; the extra work done for the uncommon operation pays off many times over in the hot (read) path. 

#### Other details

One topic that the above description has avoided thus far is updates to the table. A certain amount of mutual exclusion is required to update a hash table like this even in the absence of "relativistic" techniques; only one thread should be trying to update the head pointer in any given hash bucket at a time, for example. In general, updates must also be synchronized with resize operations in the same ways; a thread making a change to the table will simply have to wait until the resize operation is complete. 

That said, there are some tricks that can be employed to speed up these operations while a resize is going on. For a shrinking operation, update access to each hash bucket can be allowed as soon as the splicing of the lists for that bucket is complete. Expansion is, once again, a bit more complicated. Insertion at the head of the list can be done concurrently once the new table is set up since the unzipping process will not change the head pointer. Removal, though, requires coordination with the resizer to avoid removing an item that the resizer is also operating on. 

For a lot more detail on everything described here, see the original paper. 

That paper contained code implementing the relativistic hash table concept in the kernel. But, for some reason, none of this code was proposed for inclusion into the mainline. So the technique languished for nearly four years until Thomas put together his implementation, which was initially targeted at the hash table used to track netlink sockets in the kernel. Now that this infrastructure is in place, chances are that its use will expand into other areas relatively quickly.
