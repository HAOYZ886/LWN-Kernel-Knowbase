---
title: Protecting /proc/slabinfo
url: https://lwn.net/Articles/432234/
date: "March 9, 2011"
category: Security
author: "By Jake Edge March 9, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
March 9, 2011

User-space access to internal kernel information is always something of a balancing act. That information can be useful for debugging or diagnosing problems, but it can also be used by attackers to simplify exploiting security vulnerabilities. At first glance, protecting `/proc/slabinfo` so that it can't be read by non-root users seems like a reasonable restriction to help reduce targeted heap corruption exploits, but as a [patch](<https://lwn.net/Articles/431818/>) to do so was discussed, it became clear that there are some more subtle issues to consider. 

Memory for kernel objects is managed by the "slab" allocator, which allocates those objects into separate slabs based on their size. The kernel has several different slab allocators available, and users (or distributions) can choose the one they want to use when configuring the kernel. The exploits discussed in the thread were mostly aimed at the SLUB allocator because it mixes multiple object types of the same size into slabs. 

The output from `/proc/slabinfo` lists the various slabs in use, the size of objects that they contain, the number of objects currently used in the slab, and so on. That information can be used to manipulate the heap layout in a way that's favorable to attackers. 

Dan Rosenberg proposed changing the permissions of `/proc/slabinfo` to 0400, noting that ""nearly all recent public exploits for heap issues rely on feedback from /proc/slabinfo to manipulate heap layout into an exploitable state"". As he points out, normal users shouldn't really need access to that file, and on systems where that is desirable, the administrator can set the permissions appropriately. 

While few argued that unprivileged users need access to the information (at least by default), there were immediate questions about whether shutting off the access would really be an obstacle for attackers. Matt Mackall [put it](<https://lwn.net/Articles/432473/>) this way: 

Looking at a couple of these exploits, my suspicion is that looking at slabinfo simply improves the odds of success by a small factor (ie 10x or so) and doesn't present a real obstacle to attackers. All that appears to be required is to arrange that an overrunnable object be allocated next to one that is exploitable when overrun. And that can be arranged with fairly high probability on SLUB's merged caches. 

That "10x" factor is important. If it were several order of magnitudes higher, it would be clearer that possibly inconveniencing some users is worthwhile. But making an attacker only work ten times harder may not be worth it as Mackall [observes](<https://lwn.net/Articles/432480/>): 

There are thousands of attackers and millions of users. Most of those millions are on single-user systems with no local attackers. For every attacker's life we make trivially more difficult, we're also making a few real user's lives more difficult. It's not obvious that this is a good trade-off. 

Mackall describes the basic idea behind these "heap smashing" attacks well, but Rosenberg gives a [more detailed description](<https://lwn.net/Articles/432479/>) of how they work: 

The most common known techniques involve overflowing into an allocated object with useful contents such as a function pointer and then triggering these (various IPC-related structs are often used for this). It's also possible to overflow into a free object and overwrite its free pointer, causing subsequent allocations to result in a fake heap object residing in userland being under an attacker's control. 

So an attacker that can observe `/proc/slabinfo` can gain a great deal of information about slabs that would allow them to arrange the situation they need on the heap. But in the discussion, it became clear that there are other ways to gain some of that information (via observing `/proc/meminfo` for instance), though it turns out that an attacker can rely on probability to arrange objects in the "right" order. 

One could, as Mackall suggested, pre-allocate a large number of objects such that a new page is allocated to the slab. The contents of that page are then largely under the control of the attacker, so freeing one of those objects and allocating the other kind will very likely result in the needed arrangement. 

That led Pekka Enberg to [propose](<https://lwn.net/Articles/432483/>) a patch that would randomize the object layout within a slab. The idea is that attackers couldn't depend on the current sequential allocation of objects in the slab, as the free list in a new slab would be in a random order. When freeing one object and then allocating another, there would be no guarantee that the two would be sequential. That approach, too, falls by the wayside because of probability. 

Even with a randomized slab, an attacker can half fill the slab page with overrunnable objects, then allocate an exploitable object; it will then have a roughly 50% chance of being in the right place. One could also fill the slab page with overrunnable objects, then free an object--or every tenth object. The holes left behind will have a high probability being in the right place. Essentially, Mackall and others in the thread showed that the current attacks using `/proc/slabinfo` output were insufficiently imaginative. 

Mackall [noted](<https://lwn.net/Articles/432488/>) that the real underlying problem is that it is too easy for programmers to copy the wrong amount of data from user space (which is how most of these object overruns occur). He suggested that a `copy_from_user()` interface that was harder to get wrong might help reduce these kinds of problems: 

I think the real issue here is that it's too easy to write code that copies too many bytes from userspace. Every piece of code writes its own bound checks on copy_from_user, for instance, and gets it wrong by hitting signed/unsigned issues, alignment issues, etc. that are on the very edge of the average C coder's awareness. 

We need functions that are hard to abuse and coding patterns that are easy to copy, easy to review, and take the tricky bits out of the hands of driver writers. 

There was general agreement that better interfaces should be added to reduce these kinds of problems. Alan Cox [mentioned](<https://lwn.net/Articles/432494/>) that Arjan van de Ven had created some ""copy_from_user validation code [that] already does verification checks on the copies using gcc magic"". While Rosenberg is still interested in pursuing the `/proc/slabinfo` protection along with Enberg's slab randomization patch, it's not clear that there will be much support for it from other kernel developers. Given that it imposes a performance penalty along with potentially inconveniencing users, without any real benefit, it may be rather hard to sell.
