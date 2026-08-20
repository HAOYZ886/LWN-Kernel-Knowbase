---
title: Repealing the poll() tax
url: https://lwn.net/Articles/758324/
date: "June 27, 2018"
category: Performance regressions; poll
author: "By Jonathan Corbet June 27, 2018"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 27, 2018

One of the new features merged for the 4.18 kernel is [a new polling interface](<https://lwn.net/Articles/743714/>) using the asynchronous I/O mechanism. As part of this work, the internal implementation of how the various polling-related system calls (`poll()`, `select()`, and `epoll_wait()`) work was significantly changed. The reporting of a significant performance regression has now put all of that work into doubt, though. While it could be reverted, the more likely outcome would appear to be another set of changes to how polling works in the kernel. 

As a reminder, kernels prior to 4.18 expect filesystems and device drivers to provide a single `poll()` method in the `file_operations` structure: 

```
int (*poll) (struct file *file, struct poll_table_struct *table);
```

This function's job is twofold: add the wait queue(s) on which I/O-readiness events may be reported to `table`, and return a bitmask describing the I/O operations that could be performed immediately without blocking. In 4.18, these tasks have been split out into separate methods: 

```
struct wait_queue_head *(*get_poll_head)(struct file *file, int mask);
        int (*poll_mask) (struct file *file, int mask);
```

Now `get_poll_head()` returns a single wait queue on which events will be reported, while `poll_mask()` indicates the operations that can be performed immediately. The old `poll()` interface remains in the kernel because many drivers have not been converted, but the long-term intent is to get rid of it. 

On June 22, a problem with this new interface came to light in the form of [a performance-regression report](<https://lwn.net/Articles/758329/>) from the kernel test robot. In particular, a test that exercises `poll()` heavily regressed by 8.8%, which is a significant performance hit. Linus Torvalds was quick to [put his finger on the problem](<https://lwn.net/Articles/758330/>): the new polling interface is slowing things down. In particular, the replacement of the single invocation of `poll()` with calls to two other methods added another indirect call to the polling path. 

An additional function call may not seem like that heavy a cost, but indirect calls (where the address of the function to be called is computed at run time) are relatively expensive. The advent of Spectre has made that situation worse, since indirect calls must use retpolines on affected processors; that makes them quite a bit more expensive than they were before. So the new scheme has made polling significantly more expensive and, since polling is a performance-critical operation in many workloads, that is a real problem. 

Torvalds was unimpressed with the changes in general; he said that he was ""inclined to just revert the whole mess"". Christoph Hellwig [responded](<https://lwn.net/Articles/758333/>) with a quick patch that attempted to eliminate some of the extra overhead, but that didn't win applause from anybody. Some more serious changes were clearly called for. 

The direction of the most likely fix was suggested by Torvalds in that same message. The introduction of `get_poll_head()` already limits drivers to using a single wait queue to signal I/O-related events — a change that is not universally popular, but it is only a problem for a small number of drivers. Rather than provide a callback to obtain a pointer to that queue, Torvalds suggested, that queue pointer could just be stored in the `file` structure, where it would be immediately available when needed. 

Hellwig noted that this solution would not work for every case: 

People are doing weird things with their poll heads, so we can't do that unconditionally. We could however offer a waitqueue pointer in struct file and most users would be very happy with that. 

One of those cases turned out to be in the networking code, and in the ability to perform busy waiting in particular. Hellwig ended up reworking some of that code before writing a patch adding a new field: 

```
struct wait_queue_head  *f_poll_head;
```

to `struct file` and removing the `get_poll_head()` method entirely. The entire set of patches has been posted in [Hellwig's Git tree](<http://git.infradead.org/users/hch/vfs.git/shortlog/refs/heads/remove-get-poll-head>). 

The changes simplify the polling code somewhat, and they should remove the 4.18 performance regression (though no benchmark results confirming that have been posted yet). The cost comes in the form of adding another pointer to `struct file`, of which there can be many instances on a busy system. The fixes are also the sort of change that is normally not seen as desirable after the close of the merge window, and the networking changes have not yet been approved by the networking developers. 

An argument could thus be made in favor of reverting the polling changes entirely and trying again in 4.19. That may be exactly what happens if the networking developers resist the changes in their subsystem. The more likely outcome, though, is that these changes will receive whatever additional fixes prove to be necessary and will be merged in the near future. The new polling mechanism brings significant performance improvements for users who can take advantage of the asynchronous I/O interface, and they would prefer not to wait for it if possible.
