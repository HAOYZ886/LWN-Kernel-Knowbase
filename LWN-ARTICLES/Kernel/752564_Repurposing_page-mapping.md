---
title: "Repurposing page->mapping"
url: https://lwn.net/Articles/752564/
date: "April 26, 2018"
category: "Memory management-struct page"
author: "By Jonathan Corbet April 26, 2018 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 26, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

The [`page` structure](<https://lwn.net/Articles/565097/>) is one of the most complex in the kernel due to the need to cram the maximum amount of information into as little space as possible. Each field is so heavily overloaded that developers prefer to avoid making changes to `struct page` if they can avoid it. That didn't deter Jérôme Glisse from proposing a significant change during two plenary sessions at the 2018 Linux Storage, Filesystem, and Memory-Management Summit, though. There are some interesting benefits on offer, but getting there will not be a simple task. 

The `mapping` field of `struct page` describes where the page came from. For page-cache pages, `mapping` points to the `address_space` structure identifying the file the page belongs to; [![\[Jérôme Glisse\]](https://static.lwn.net/images/conf/2018/lsfmm/JeromeGlisse-sm.jpg)](<https://lwn.net/Articles/752579/>) anonymous pages use `mapping` to map back to the `anon_vma` structure. For pages used by the kernel itself, the `mapping` field can be used by the slab allocator. Like everything else in this structure, `mapping` is a complicated field with a number of different interpretations depending on how the page is being used at any given time. 

Glisse has his own designs on that field, but first he must find a way to eliminate its current uses. Most of the time, code that is working with a `page` structure has found that structure by way of a virtual memory area (VMA) or a `file` structure; in either case, the mapping information can be obtained from those structures. In the contexts where that information is available, there is no need to store it in the `page` structure itself; it can be replaced by changing interfaces to pass the mapping information down the call chain. Doing so allows him to eliminate most uses of `mapping` and use that space for other purposes. 

In particular, he is looking at using that field to attach a structure for threads that are waiting on the page. Currently, waiting on specific pages is done with a set of 256 shared wait queues; replacing those queues would make the wakeup process faster in cases where the queues get long. In the normal case, when nobody is waiting on a page, the `mapping` field would point to a structure like: 

```
struct page_mapping {
            struct address_space *mapping;
    	unsigned long flags;
        };
```

Essentially, this mechanism is adding a layer of indirection for access to the mapping information, and adding some flags for good measure. When somebody needs to wait on that page, though, this structure would be replaced with: 

```
struct page_wait {
            struct wait_queue_head wq[PAGE_WRITABLE_BITS];
    	struct wait_queue_entry entry;
    	struct page_mapping base;
    	spinlock_t lock;
    	bool own;
        };
```

When this substitution is done, the pointer to the `page_wait` structure has its lowest significant bit set to flag the change. Any code needing the old `mapping` field would need to notice that change and follow the pointers through another level of indirection to get that information. This situation would persist until the last waiter is removed; at that point, the pointer to the `page_mapping` structure would be restored. 

Hugh Dickins jumped in to ask Glisse what problem he was trying to solve with this change. Glisse responded that the point was to address the length of the shared wait queues, which have been growing over time. Dickins said that the problem could be solved more simply by just adding more queues. Kent Overstreet said that perhaps [rhashtables](<https://lwn.net/Articles/751374/>) could be used for this purpose instead. He is fully in favor of eliminating the `mapping` field, he said, but it would be a shame to replace that field with something else. But Glisse said his goal is not to fully remove the field; he really just wants to make it easier to attach structures to the `page` structure. 

Matthew Wilcox suggested that perhaps the `private` field could be used for this kind of structure attachment, but it seems that there are already too many other uses of that field. Chris Mason asserted forcefully (and humorously) that "`private` is mine!". 

Dickins said that there could be some value in generalizing the `mapping` field, but that using it for waiting on pages in particular is "a bit peculiar". Glisse responded that this functionality comes "almost for free" with little code required. But Dickins insisted that `mapping` says a lot about the identity of any given page; if it is replaced with something else, the page loses that identity information. He described this mechanism as an "odd misuse" to add some transient information. 

Rik van Riel asked about how synchronization between page-lookup and truncate operations is handled in this scheme. Truncation needs `mapping`, Glisse said, but it's one of only a few places in the kernel that do. Making things work is simply a matter of adding awareness of the new scheme in those places. Dave Chinner said that XFS checks for a null `mapping` value to handle truncation; that will break in the new scheme. Glisse suggested putting in a new helper, but Overstreet said that, instead, the time has come to find a better way to handle locking around truncate operations. 

Dan Williams repeated the question of why this pointer was really needed; this time Glisse said that he needs it for page write protection, a topic slated to be discussed on the following day. He needs to set a pointer inside of `struct page`, and `mapping` happens to be the easiest one to grab. 

Glisse concluded the session by acknowledging that most developers seemed to feel that this change "looks ugly". He will post the patches in the near future anyway, though, and see what the reaction is at that time. 

#### Generic write protection

The group was not yet done with `mapping`, though; Glisse led another session on the topic in a plenary session the following day, where he delved deeper into the motivation for this work. There are a number of situations where nominally writable memory must be globally write-protected for a time. One possible use case is "kernel duplicated memory", where memory pages are duplicated across the system for performance. In a system with multiple GPUs, it can be worthwhile to have duplicate copies of input data in multiple pages so that each GPU can access that data with its full available bandwidth. Another is PCIe atomic transactions — 256-byte transactions that must wait for an acknowledgment from the controller, which can be a slow process. It can be made quite a bit faster, though, if the memory is write-protected on the host; the GPU (or other remote processor) can then do its work without using slow atomic operations. 

Also, in general, he wanted to minimize the reliance on the `mapping` field and open up some space within the `page` structure for other uses. 

Getting there requires putting together a comprehensive picture of where `mapping` is used now and which alternatives may exist. For example, file-related system calls use it, but they also are given the file (and thus the mapping) as a parameter, so they don't need to obtain it from the `page` structure. Similarly, memory-related system calls have access to the virtual memory area (VMA) being operated on; the mapping can [![\[Jérôme Glisse\]](https://static.lwn.net/images/conf/2018/lsfmm/JeromeGlisse2-sm.jpg)](<https://lwn.net/Articles/752803/>) be found there. A bit trickier might be getting at the mapping information from the BIO structures used for block I/O. Glisse said that he expects the relationship between a BIO and the mapping to be unchanging, but a BIO can actually contain pages from multiple files at times. It looks like the problem can be solved, but it may involve storing the mapping information in the BIO structure directly. 

Once most users of `mapping` have been redirected, that field can be replaced with a pointer to a new structure, adding a level of indirection. The kernel same-page merging (KSM) mechanism does that already, as it turns out. So the first step might be to have `mapping` point to a `page_attachment` structure (different from the `page_mapping` structure shown the day before) that would replace the KSM mechanism and make it more generic. 

Boaz Harrosh repeated the question from the day before about why `mapping` is being targeted rather than `private`. Glisse responded that `private` is used in "funny ways"; it's not always easy to know what is going on. It is much easier to just remove the users of `mapping`. 

Dickins agreed that `mapping` is the right field to target. But he suggested that, if it is used in so few places, why not just put the replacement structure in a stable location rather than attaching it to the `page` structure? The answer seems to be that it is never clear how long a particular structure will end up being attached to the `mapping` field. But Dickins insisted that it could make sense to have `mapping` point to an always-useful structure. He (again) said that `mapping` tracks the identity of a page, so it should be replaced by a structure that still handles that function. 

When the [`anon_vma` mechanism](<https://lwn.net/Articles/86715/>) was added (to allow the mapping of anonymous pages back to the tasks that reference them), he continued, the use of the `mapping` field was "fudged" to accommodate it. KSM then fudged it some more. Developers have always wanted to avoid changing every filesystem in the kernel, so they have taken the easy route out. But, he said, if Glisse is willing to do the work to change this field entirely, he should get rid of the "peculiarity" surrounding it. 

David Howells asked if, instead, the `address_space` structure could be eliminated entirely, with the necessary information being put back into the `inode` structure. Glisse said that he would like to see that happen, but that the prospect of doing it is daunting. His plan is to create the feature he wants first, and maybe look at bigger tasks like this later. 

#### The path forward

Glisse laid out his plan for upstreaming this work; it is designed to maximize the confidence in the changes before depending on them. The first step would be to replace every reference to `mapping` with a call to a helper function. Then the low-level functions that use `mapping` will be modified to take that information as a parameter. This change will be done with a tool like Coccinelle, and the value of the parameter would be `NULL` at the outset; the code would continue to use the `mapping` field and would ignore the new parameter. 

In a subsequent development cycle, filesystems would be changed to pass the mapping information down to where it's needed; the memory-management system calls would see similar changes. At some point in this process, the KSM mechanism would be converted to the more generic approach. 

Once this work is complete, it should be possible to avoid using the `mapping` in almost all situations. But, for a couple of releases, the code would continue to use `mapping` while checking to ensure that it matches the value passed down the call stack. This should build confidence that the conversion has been done properly — or point out the places where it has not. Once that confidence reaches a sufficiently high level, the final step could be taken and the `mapping` field would no longer be used. 

Harrosh said that there may well be places where the two mapping values do not match. That could be the result of bugs, or places where, for whatever reason, the real mapping is not the one stored in the `page` structure. But Glisse plans to put tests in the places where the `mapping` field is actually used and, thus, is expected to be valid. 

Dickins worried that this work sounds like a large amount of churn; he said he would need to know more about the benefits that will result. It would, he said, require "a resounding 'yes'" from the filesystem developers. Mason said that making it easier to share pages between files is "a big deal", so this is an important feature for him. As the subsequent discussion showed, though, there are a number of challenges to be overcome before it becomes possible to share pages between files; that topic was eventually set aside as something for the filesystem developers to work out at some other time. 

Dickins also suggested that, if the real objective is to globally write-protect pages, a new page flag might be a better solution to the problem. Williams, instead, said that the real problem is user space writing to pages when they need to be exclusively owned by a device like the GPU. This solution might be seen as surrendering to bad behavior from user space, when the right thing to do is to push back and say "don't do that". Glisse replied that changing user space is not possible; it may be a ten-year-old program using a library that has been updated to accelerate operations with the GPU. A solution to that problem, Williams said, is to require applications to migrate to a newer library if they want the newer performance options. 

Josef Bacik said that the proposed mechanism solved a mapping-related problem that he had run into, so he would be happy to see it go in. Overstreet agreed that a number of use cases would benefit from the change. Dave Hansen said that the benefits could go beyond GPUs to other devices that have their own memory. Dickins said that he had no reservations about the objective, but he still wasn't sure that changing `mapping` was the right way to get there. But he got the sense that the filesystem people were glad to have a sucker who is willing to do this work, and that they would prefer that the memory-management people not obstruct it. 

Wilcox noted that some of these problems have been solved before; SGI was doing binary text replication years ago. That prompted Johannes Weiner to observe that, if the group has begun trading SGI anecdotes, it must be time to wrap up the discussion.
