---
title: Two PaX features move toward the mainline
url: https://lwn.net/Articles/668876/
date: "December 23, 2015"
category: "Security-Kernel hardening"
author: "By Jake Edge December 23, 2015"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
December 23, 2015

As the [Kernel self-protection project](<http://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project>) (KSPP) ramps up in the month and a half since its [formation](<https://lwn.net/Articles/663361/>), several features from the [PaX project](<https://pax.grsecurity.net/>) are starting their journey toward the mainline. The reception on the [kernel-hardening mailing list](<http://www.openwall.com/lists/kernel-hardening/>) that is being used to coordinate KSPP work has been positive, but the real test for these features will come when they are proposed for the mainline. Two specific patch sets have been posted recently, for PAX_REFCOUNT and PAX_MEMORY_SANITIZE, that we will look at here. 

#### PAX_REFCOUNT

The idea behind the [PAX_REFCOUNT patch set](<https://lwn.net/Articles/668724/>), posted by David Windsor, is to detect and handle overflows in reference-count variables. The kernel uses reference counts to track objects that have been allocated, incrementing or decrementing the count as references to them come and go; the kernel frees those objects when the count reaches zero. But if there is a path in the kernel where the count doesn't get decremented when an object reference gets dropped, an attacker could use that path to overflow and wrap the reference counter, effectively setting it to zero when there are actually still valid references to it. The object will be freed, but will still be used by those with references, leading to a use-after-free vulnerability. 

This is not the first attempt to add this kind of overflow protection to the kernel. But when Windsor [posted about a related idea](<http://thread.gmane.org/gmane.linux.ubuntu.hardened.general/574>) for `kref`, which is a kernel abstraction for reference counts, back in 2012, the idea ran aground on how it handled the overflows. Like the original PaX patches, Windsor's [patch](<http://thread.gmane.org/gmane.linux.ubuntu.hardened.general/574/focus=1257934>) would call `BUG_ON()` for reference counts that reached `INT_MAX`, instead of incrementing them. That would crash the kernel if the count ever reaches `INT_MAX`, which Greg Kroah-Hartman [objected to](<http://thread.gmane.org/gmane.linux.ubuntu.hardened.general/574/focus=1257948>): 

So you are guaranteeing to crash a machine here if this fails? And you were trying to say this is a "security" based fix? 

And people wonder why I no longer have any hair... 

But as Windsor and others pointed out, there is no sensible recovery that can be done if a reference count is about to wrap. An alternative might be to simply not change the counter (and put a warning into the kernel log) once it reaches `INT_MAX`, but that would lead to a memory leak. Overall, at least at the time, Kroah-Hartman was clearly skeptical of the whole idea—or even that a `kref` wrap could be exploited. However, Kees Cook did [describe](<http://thread.gmane.org/gmane.linux.ubuntu.hardened.general/574/focus=1257985>) the way an exploit might work: 

Based on what I've seen, the "normal" exploit follows this pattern: 

user1: alloc(), inc  
user2: inc  
user2: fail to dec  
*repeat user2's actions until wrap*  
user3: inc  
user3: dec, free()  
user1: operate on freed memory zomg  

In the recent posting of PAX_REFCOUNT, Windsor has essentially broken up the PaX project's patches and applied them to the 4.2.6 stable kernel, though he is working on rebasing on linux-next. He noted a [post](<https://forums.grsecurity.net/viewtopic.php?f=7&t=4173>) on the grsecurity forums where the feature is well documented. The implementation changes the kernel's operations on `atomic_t` types so that overflows cannot occur; increments beyond `INT_MAX` are disallowed. In addition, processes that would have caused an overflow are sent a `SIGKILL` so that they can do no further damage. Windsor suggested that the signal might be too severe to start with: 

When an overflow is detected, SIGKILL is sent to the offending process. This may be too drastic for an initial upstream submission. WARN_ON may be more appropriate until distros have some time to absorb it and report any unaddressed overflows. 

The patches also create an `atomic_unchecked_t` type that acts just like today's `atomic_t`; it does no checking for overflow. In fact, the bulk of the patches are to various subsystems that use atomic variables but don't use them as reference counts; they are switched to use the new unchecked type. If the patches get merged, new users of atomic variables will need to determine if they are being used as reference counts or not to choose the proper atomic type. 

So far, the comments on the patches have been light, but one suspects the code churn needed to switch all of those atomic types will bring some complaints when the patches get posted more widely. One could imagine creating a new type for those variables that need the checking, but that would require constant vigilance to ensure that any reference counts added to the kernel actually used the new type. That problem still exists with the posted patches, however, since new `atomic_unchecked_t` variables will need to be scrutinized to see that they aren't being used as reference counts. 

#### PAX_MEMORY_SANITIZE

One way to mitigate the effect of use-after-free vulnerability or to stop various information leaks is to "sanitize" memory that is being freed by writing zeroes or some other constant value to it. That is the idea behind the [PAX_MEMORY_SANITIZE](<https://en.wikibooks.org/wiki/Grsecurity/Appendix/Grsecurity_and_PaX_Configuration_Options#Sanitize_all_freed_memory>) feature. Laura Abbott posted a [partial port of the feature](<https://lwn.net/Articles/668850/>) to kernel-hardening on December 21\. 

In particular, Abbott's patches add the sanitization to the slab allocators (slab, slob, and slub), but not for the buddy allocator as the full PAX_MEMORY_SANITIZE feature does. That means ""that allocations which go directly to the buddy allocator (i.e. large allocations) aren't sanitized"". The actual sanitization is done using a fixed value (0xff for all architectures except x86-64, which uses 0xfe) that is written over the entire object before it is freed. Abbott plans to look into adding sanitization to the buddy allocator sometime in the new year. Another change that Abbott made to the PaX version of the feature was to add an option to handle the sanitization in the slow path of the allocator. 

Christoph Lameter [complained](<https://lwn.net/Articles/668962/>) that the feature was similar to the slab-poisoning feature, so it should use that mechanism instead. Abbott [agreed](<https://lwn.net/Articles/668963/>) that the features were similar, but said that poisoning is a debug feature and this work is targeting kernel hardening so ""it seemed more appropriate to keep debug features and non-debug features separate hence the separate option and configuration"". 

The cost of sanitization is performance, of course. Abbott said she measured impacts of 3-20% depending on the benchmark. But the impact of compiling the feature into the kernel, but turning it off at runtime (using the `sanitize_slab=off` boot option), is negligible. 

Lameter also suggested using the `GFP_ZERO` flag to make allocations be zeroed before being returned. If there were a mode that set that flag for all allocations it would provide ""implied sanitization"". But doing it that way would move the performance impact from the free path to the allocation side, which is typically more performance sensitive, as Dave Hansen [pointed out](<https://lwn.net/Articles/668969/>). It also means that unallocated memory would still store the potentially sensitive contents of the previous object until it is allocated again. 

Instead of writing the fixed sanitization value across the object, writing zeroes would potentially allow the allocation path to skip the zeroing step, Hansen [suggested](<https://lwn.net/Articles/668970/>). That might reduce some of the performance impact, though doing the zeroing at allocation time does leave the object's memory cache-hot, as Lameter [noted](<https://lwn.net/Articles/668972/>). But zeroing has another downside that Abbott [mentioned](<https://lwn.net/Articles/668973/>): 

poisoning with non-zero memory makes it easier to determine that the error came from accessing the sanitized memory vs. some other case. I don't think the feature would be as strong if the memory was only zeroed vs. some other data value. 

Overall, both patches were fairly well-received, but the hardening list is likely made up of those who are predisposed to look favorably on these kinds of changes. Based on [discussions](<https://lwn.net/Articles/662907/>) at last year's Kernel Summit, mainline developers should in theory be more receptive to patches that seek to mitigate whole classes of security bugs. If these PaX features can get merged eventually, there are some even more intrusive ones that could also attempt to run the gauntlet of the linux-kernel mailing list. Just where the line is—or even if there is one—is still unclear, but patches like these may help define it.
