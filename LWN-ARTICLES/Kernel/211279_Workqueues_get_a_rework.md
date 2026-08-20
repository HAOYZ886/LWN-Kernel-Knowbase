---
title: Workqueues get a rework
url: https://lwn.net/Articles/211279/
date: "November 27, 2006"
category: Workqueues
author: 
---

The [workqueue mechanism](<http://lwn.net/Articles/23634/>) allows kernel code to defer processing to a later time. Workqueues are characterized by the existence of one or more dedicated processes which execute queued jobs; since work is done in process context, it can sleep if need be. Workqueues can also delay the execution of specific jobs for a caller-specified period. They are used in many places throughout the kernel. 

David Howells recently took a look at workqueues and noticed that the `work_struct` structure, which describes a task to be executed, is rather large. It can be 96 bytes on 64-bit machines. That is fairly heavy for structures which can be used in reasonably large quantities. So he set out to find ways to make it smaller. He succeeded, but at the cost of some changes to the workqueue API. 

The causes of bloat in `struct work_struct` are: 

  * The timer structure embedded in each one. Many users of workqueues never need the delay feature, but every queued bit of work carries along a `timer_list` structure, just in case. 

  * The private data pointer, which is passed to the actual work function. Many work functions use that pointer, but it can often be calculated from the `work_struct` pointer using `container_of()`. 

  * An entire word is used to store a single bit: the "pending" flag which indicates that a `work_struct` is currently in a queue waiting to be executed. 

David addressed each of these issues. As a result, there are now two types of work structure (`struct work_struct` and `struct delayed_work`); the timer information has been removed from the former. The private data pointer is gone; work functions instead get a pointer to the associated `work_struct` (or `delayed_work`) structure. And some internal trickery was used to get rid of the word holding the "pending" bit. 

The result of these changes is that almost every part of the workqueue API has changed. There are now two ways of declaring a workqueue entry: 

```
typedef void (*work_func_t)(struct work_struct *work);
    
        DECLARE_WORK(name, func);
        DECLARE_DELAYED_WORK(name, func);
```

The prototype for the work function has changed; it is now a pointer to the relevant work queue entry. Note that a `work_struct` pointer is always passed, even in the case of delayed work. It would appear that the programmer is expected to count on the fact that `struct work_struct` is the first field of `struct delayed_work`, so `container_of()` should work as expected. As long as nobody rearranges `struct delayed_work`, anyway. 

For work structures which must be set up at run time, the initialization macros now look like this: 

```
INIT_WORK(struct work_struct work, work_func_t func);
        PREPARE_WORK(struct work_struct work, work_func_t func);
        INIT_DELAYED_WORK(struct delayed_work work, work_func_t func);
        PREPARE_DELAYED_WORK(struct delayed_work work, work_func_t func);
```

The `INIT_*` versions initialize the entire structure; they must be used the first time a structure is initialized. Thereafter, the `PREPARE_*` versions, which are slightly faster, can be used. 

The functions for adding entries to workqueues (and canceling them) now look like this: 

```
int queue_work(struct workqueue_struct *queue,
                       struct work_struct *work);
        int queue_delayed_work(struct workqueue_struct *queue,
                               struct delayed_work *work);
        int queue_delayed_work_on(int cpu,
                                  struct workqueue_struct *queue,
                       	      struct delayed_work *work);
        int cancel_delayed_work(struct delayed_work *work);
        int cancel_rearming_delayed_work(struct delayed_work *work);
```

Interestingly, David has added a variant on the workqueue declaration and initialization macros: 

```
DECLARE_WORK_NAR(name, func);
        DECLARE_DELAYED_WORK_NAR(name, func);
        INIT_WORK_NAR(name, func);
        INIT_DELAYED_WORK_NAR(name, func);
        PREPARE_WORK_NAR(name, func);
        PREPARE_DELAYED_WORK_NAR(name, func);
```

The "NAR" stands for "non-auto-release." Normally, the workqueue subsystem resets a work entry's pending flag prior to calling the work function; that action, among other things, allows the function to resubmit itself if need be. If the entry is initialized with one of the above macros, however, this reset will not happen, and the work function is expected to reset the flag itself (with a call to `work_release()`). The stated purpose is to prevent the workqueue entry from being released before the work function is done with it - but there is nothing in the clearing of the pending bit which would cause that release to happen. Perhaps that is why there are no users of the `_NAR` variants in David's patch. It may be that somebody is thinking about implementing reference-counted workqueue structures in the future. 

Meanwhile, these changes require a lot of fixes throughout the kernel tree; that drew [a complaint](<https://lwn.net/Articles/211305/>) from Andrew Morton, who was unable to make those changes mesh with all of the other patches queued up for the opening of the 2.6.20 merge window. Andrew suggested that the workqueue patches could be merged after 2.6.20-rc1 comes out, as was done with the interrupt handler function prototype in 2.6.19. But Linus, who likes the workqueue patches, [would rather get them in sooner](<https://lwn.net/Articles/211306/>): 

I'd actually prefer to take it before -rc1, because I think the previous time we did something after -rc1 was a failure (the whole irq argument handling thing). It just exposed too many problems too late in the dev cycle. I'd rather have the problems be exposed by the time -rc1 rolls out, and keep the whole "we've done all major nasty ops by -rc1" thing. 

So it seems that, somehow, all of the pieces will be made to fit and the workqueue API will change in 2.6.20.
