---
title: Lockless MPSC FIFO queues for io_uring
url: https://lwn.net/Articles/1081871/
date: "July 15, 2026"
category: "Lockless algorithms; Releases-7.2; io uring"
author: "By Jonathan Corbet July 15, 2026"
---

> **For humans, by humans**
> 
> Every article on LWN.net is written for humans, by humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the slop at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 15, 2026

Processes that use [io_uring](<https://man7.org/linux/man-pages/man7/io_uring.7.html>) tend to keep a lot of balls in the air; being able to have many operations underway at any given time is part of the point of that API in the first place. The io_uring subsystem must, as a result, keep track of a lot of tasks that have to be performed at the right time. In current kernels, io_uring uses a standard kernel linked-list primitive to track those work items. As of the 7.2 kernel release, though, io_uring will, instead, use a new lockless, multi-producer, single-consumer (MPSC) queue, resulting in some notable performance gains. Lockless algorithms tend to be tricky, but the one used here is relatively approachable and shows how these algorithms can work. 

#### The old way's shortcomings

Task queues in pre-7.2 io_uring are based on the kernel's [lockless singly linked list](<https://elixir.bootlin.com/linux/v7.1.2/source/include/linux/llist.h>) (llist) API. At the core of this type is a simple structure: 

```
struct llist_node {
    	struct llist_node *next;
        };
```

This structure, when embedded within another structure containing the actual data of interest, contains the links that bind those outer structures into a list. 

There are a few reasons why this list type, despite being designed for performance, is not ideal for io_uring. Since an llist is a singly linked list, it can only realistically be accessed from the head. For a list where producers add items and consumers remove them, an llist is essentially a stack. Work items in io_uring need to be processed in the order they were received, for basic fairness purposes if nothing else, so a pass must be made over the task queue to reverse its order before each processing run. To make things worse, io_uring might choose not to process the whole list if it is too long, but the remaining items, having been reversed in order, cannot simply be put back onto the task list. So a separate list for reversed-but-not-processed items must be maintained. Finally, adding items to an llist involves accessing a single head pointer; that can be done without taking locks, but it does require a retry loop. For heavily contended lists, those retries (and associated cache-line bouncing) can hurt. 

Solving these problems requires a data structure that is better suited to the needs of the io_uring subsystem. It must handle multiple producers putting work items on the lists, without locking and with a minimum amount of contention. There is a single consumer for each list; that consumer should avoid cache contention with the producers to the greatest extent possible. And, clearly, the need to reorder lists before processing should not exist. 

#### Lockless MPSC queues

The solution is the lockless MPSC queue, [posted](<https://lwn.net/ml/all/20260612025125.1690253-1-axboe@kernel.dk>) by Jens Axboe and using an algorithm credited to Dmitry Vyukov. This queue still uses `struct llist_node` to tie the entries of the list together, but the head of the list looks like this: 

```
struct mpscq {
    	struct llist_node	*tail;
    	struct llist_node	stub;
        };
```

The term "head" is actually a bit misleading, since there is no pointer to the head of the list here; we'll get to that later. This view of the list is intended for the use of the producers, who need to add items to the tail of the list. The `stub` entry is a sentinel that is only present on the list if it is the only entry there — if the list is empty, in other words. When the list is initialized, the `tail` pointer is set to point at the stub entry. 

> ![\[Empty queue\]](https://static.lwn.net/images/2026/mpscq1.svg)

The stub entry's `next` pointer is set to `NULL`. The addition of a node to the tail of the list is done with a call to this short function: 

```
static inline bool mpscq_push(struct mpscq *q, struct llist_node *node)
        {
    	struct llist_node *prev;
    
    	node->next = NULL;
    	prev = xchg(&q->tail, node);
    	WRITE_ONCE(prev->next, node);
    	return prev == &q->stub;
        }
```

The `next` pointer in the new entry is set to `NULL`, indicating that it is the end of the list. Then, the `xchg()` call atomically stores a pointer to the new entry in the list's `tail` pointer, returning the previous value of that pointer; in the case of an empty list, that will be a pointer to the stub entry. The `next` pointer of the previous end-of-list entry (which, again, might have been the stub) is then set to the new entry, completing the task of adding that entry to the list. 

> ![\[Queue with one entry\]](https://static.lwn.net/images/2026/mpscq2.svg)

There is a subtlety to lockless algorithms that is worth noting here. Once the `tail` pointer has been aimed at the new list entry, that entry is visible to the rest of the world. Among other things, that visibility implies that the `next` pointer in the new entry must be properly set before changing `tail`. Normally, either the compiler or the CPU could feel entitled to reorder the assignments to `next` and `tail`, which could result in the new `tail` being visible before the new node is fully initialized. The `xchg()` operation, though, is defined as a full barrier, meaning that operations that happen prior (such as the assignment to `next`) must be visible to the rest of the system before the exchange can take place. In the absence of a barrier operation, it would have been necessary to manually insert a barrier between those two assignments. 

If multiple CPUs attempt to add entries to the same list at the same time, the `xchg()` call will serialize them, ensuring that the `tail` pointer is updated in an orderly manner. If two CPUs perform their `xchg()` calls simultaneously, one will "win" and go first, followed by the other. That could result in a list that, briefly, looks like this: 

> ![\[Queue during contention\]](https://static.lwn.net/images/2026/mpscq3.svg)

Since each `xchg()` call returns the previous state of the `tail` pointer, each addition knows where the preceding entry in the list is. That allows it to set the `next` pointer accordingly. Once the two additions complete, the list will look like: 

> ![\[Queue after contention\]](https://static.lwn.net/images/2026/mpscq4.svg)

There is no locking required to keep the list in a coherent state, and no retry loops are needed, so addition operations are fast. Additions can be made while running within any kernel context as well. 

#### The consumer's view

The consumer side is just a little bit more involved, starting with the fact that the consumer maintains a head-of-list pointer separately from the `mpscq` structure; it is a simple `struct llist_node` pointer holding the address of the first entry in the list. The purpose of this separation is to ensure that the head and tail pointers are placed in separate cache lines, avoiding cache contention between the producers and the consumer. To remove the first entry in the list, the consumer will pass that head pointer to: 

```
static inline struct llist_node *mpscq_pop(struct mpscq *q,
    					   struct llist_node **headp);
```

There are a few cases that this function has to be prepared for. When the list is first created, in an empty state (as shown above), the head-of-list pointer will contain the address of the `stub` entry. Since additions to the list do not change the head pointer, that situation will remain until the first item is removed from the list. Imagine that no items have yet been removed from the list shown above; with the separate head pointer, the picture looks like this: 

> ![\[Queue with head pointer\]](https://static.lwn.net/images/2026/mpscq5.svg)

Item removal for that case is handled this way: 

```
struct llist_node *head = *headp, *next;
    
    	if (head == &q->stub) {
    	    head = READ_ONCE(head->next);
    	    if (!head)
    		return NULL;
    	    q->stub.next = NULL;
    	    *headp = head;
    	}
```

Remember that the addition of the first entry to the list set the `next` pointer in `stub` to that first entry; here the code checks that pointer. If it is `NULL` then the list is empty, so `NULL` is returned. Otherwise the head is advanced to the value of the stub's `next` field, which is subsequently set to `NULL`. 

> ![\[Queue with stub removed\]](https://static.lwn.net/images/2026/mpscq6.svg)

The stub has no further role in the management of the list until it is emptied again. 

Now that it has been established that there is an entry on the list, the next check is to see if it is the _last_ entry. In the negative case, when further entries exist, the head pointer can be advanced to the next of those entries, and a pointer to the head entry returned: 

```
next = READ_ONCE(head->next);
    	if (next) {
    	    *headp = next;
    	    return head;
    	}
```

After an item is returned in this way, the situation is: 

> ![\[Queue with one item removed\]](https://static.lwn.net/images/2026/mpscq7.svg)

If the `next` pointer is `NULL`, though, then there are no more entries in the list, and the `tail` pointer must, once again, be set to the stub. There is a twist, though: there may be a producer adding a new entry to the list at the same time. So a compare-and-exchange operation is needed to attempt to reset the list to the empty state: 

```
if (try_cmpxchg(&q->tail, &head, &q->stub)) {
    	    *headp = &q->stub;
    	    return head;
    	}
    	return NULL;
```

The `try_cmpxchg()` call compares the `tail` pointer to the `head` pointer (which, remember, points to the one entry in the list). If the two are equal, it atomically sets the `tail` to point to the stub, resetting the list to the empty state shown at the beginning; it then returns the last entry in the list. 

If, however, the `try_cmpxchg()` call fails, then the consumer has raced with another producer, and that producer has changed the `tail` pointer behind the consumer's back. In this case, `NULL` is returned and the last entry is left on the list until the next time the consumer retries. The consumer can distinguish this case from the list-is-empty case by seeing whether the `tail` pointer is aimed at the stub entry: 

```
static inline bool mpscq_empty(struct mpscq *q)
        {
    	return READ_ONCE(q->tail) == &q->stub;
        }
```

That describes the entire API. The code can be found in [`io_uring/mpscq.h`](<https://elixir.bootlin.com/linux/v7.2-rc1/source/io_uring/mpscq.h>); it is not, at this point, placed under `lib/` and made available to the rest of the kernel. That could, of course, be changed if an interested user outside of io_uring were to emerge. 

As of 7.2, this new queue type is used for a couple of different task lists within io_uring. The results, as described in [this patch](<https://lwn.net/ml/all/20260612025125.1690253-4-axboe@kernel.dk>), are a significant increase in performance with reduced overhead — more work is done more quickly, while simultaneously reducing the amount of time spent executing in the kernel. The io_uring code is also simplified somewhat, since it no longer needs to reverse the lists or maintain a separate list of work items that were removed from the task list but not yet acted upon. All told, it would appear to be an optimization whose time has come. 

(**Postscript** : this topic was at risk of being passed over, but it received enough votes on [the LWN public topics page](</TopicList/>), which is available to subscribers at the Project Leader level and above, that I decided to give it another look. My thanks go to the LWN readers who thought this development was worthy of an article.)
