---
title: "Page sanitization, part 2"
url: https://lwn.net/Articles/335698/
date: "June 3, 2009"
category: "Security-Kernel hardening"
author: "By Jake Edge June 3, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
June 3, 2009

Last week's Security page [looked](<https://lwn.net/Articles/334067/>) at some recently proposed patches that would "sanitize" kernel memory by clearing it as it was freed. At that time, a second version of the patches which unconditionally cleared memory when freed--dependent on the `sanitize_mem` boot parameter--was generally well received. But, perhaps folks just had not yet had a chance to look. Over the last week, multiple objections have been raised, which were mostly met with belligerent responses from developer Larry Highsmith. In many ways, this is starting to look like yet another lesson in "how not to work with the kernel community". 

The basic problem is that data can persist in memory long after that memory is freed. Sometimes that data contains passwords, cryptographic keys, confidential documents, etc., but it is impossible for the kernel to know, in the general case, which pages are sensitive. By clearing memory when it is deallocated, the lifetime of this potentially sensitive data can be reduced. A research [paper](<http://www.stanford.edu/~blp/papers/shredding.html/>) describes some experiments that showed memory values persisting for days and even weeks on Linux systems. A bug in the kernel that leaked memory information could potentially leak these values to attackers. 

So, Highsmith proposed adding a memory sanitization feature that has long been a part of the patches applied to the kernel by the [PaX security project](<http://pax.grsecurity.net/>). There is clearly a performance impact to clearing memory as it is reclaimed, but, since memory is cleared as it is allocated (to avoid obvious information leaks), the impact may not be as large as it seems at first glance. As Arjan van de Ven [points out](<https://lwn.net/Articles/335883/>): 

.. and if we zero on free, we don't need to zero on allocate. While this is a little controversial, it does mean that at least part of the cost is just time-shifted, which means it'll not be TOO bad hopefully... 

Peter Zijlstra is [concerned](<https://lwn.net/Articles/335886/>) about the cache effects: ""zero on allocate has the advantage of cache hotness, we're going to use the memory, why else allocate it. [...] zero on free only causes extra cache evictions for no gain."" But van de Ven [describes](<https://lwn.net/Articles/335889/>) how he sees the caches being affected, concluding: ""Don't get me wrong, I'm not arguing that zero-on-free is better, I'm just trying to point out that the 'advantage' of zero-on-allocate isn't nearly as big as people sometimes think it is..."" 

But some, like Alan Cox, [think](<https://lwn.net/Articles/335901/>) the performance impact is immaterial: ""If you need this kind of data wiping then the performance hit is basically irrelevant, the security comes first."" Zijlstra and others are concerned about the price that is paid by _all_ kernel users, even those who have not enabled `sanitize_mem`. He [notes](<https://lwn.net/Articles/335903/>) that the patches would add extra function calls and branches even when the feature is not enabled. Suggestions were made to benchmark the proposed code against the existing implementation, but that is where the conversation started to go off the rails. 

Highsmith obviously gets frustrated with the direction of the discussion, but rather than stepping back, he lashes out. There is certainly some provocation in the thread, Zijlstra's ""Really, get a life, go fix real bugs. Don't make our kernel slower for wanking rights."" [comment](<https://lwn.net/Articles/335914/>) certainly didn't help. But Highsmith needs to recognize that he is the one trying to get something added to the kernel, so the burden of "proof" is on him. Instead, his condescending manner seems to indicate that he feels like he is presenting the kernel community with a gift--one they are too slow-witted to understand. 

An important characteristic for kernel contributors is that they work well with the rest of the community: answer questions, respond to code review suggestions, etc. When that doesn't happen, patches tend to be ignored, regardless of their technical merit, and Highsmith seems headed down that path. When it was suggested that using `kzfree()` on specific kernel allocations for sensitive data--which would clear the memory, then free it--Highsmith [responded](<https://lwn.net/Articles/335932/>): 

That's hopeless, and kzfree is broken. Like I said in my earlier reply, please test that yourself to see the results. Whoever wrote that ignored how SLAB/SLUB work and if kzfree had been used somewhere in the kernel before, it should have been noticed [a] long time ago. 

Since Highsmith was responding to SLAB maintainer Pekka Enberg's suggestion, that response--even if true--probably wasn't the right approach. Enberg and others [asked](<https://lwn.net/Articles/335938/>) specifically about the problems in `kzfree()`, but the [response](<https://lwn.net/Articles/335940/>) from Highsmith was a combination of condescension and vagueness. As soon as Enberg and Ingo Molnar tried to pin down where those problems are, Highsmith went off on a [rant](<https://lwn.net/Articles/335944/>) about the SLOB memory allocator. 

In addition, Molnar has [pointed out](<https://lwn.net/Articles/335959/>) that some of the same sensitive values can have long lifetimes on the kernel stack: 

Long-lived tasks that touched any crypto path (or other sensitive data in the kernel) and leaked it to the kernel stack can possibly keep sensitive information there indefinitely (especially if that information got there in an accidentally deep stack context) - up until the task exits. That information will outlive the freeing and sanitizing of the original sensitive data. 

Rather than recognize this as an additional area that needs addressing, Highsmith just [continues](<https://lwn.net/Articles/335963/>) his tirade: 

But you and the other cabal of vagueness have only sent mostly useless comments, outright uncivil responses, obvious misdirection attempts, unfounded critics, etc. I haven't seen more fallacies put together since the last time I read an unreleased film script by Jerry Lewis. 

Overall, the idea of clearing memory as it is freed based on a boot time flag is reasonable. Several kernel hackers, including Cox and Rik van Riel, have expressed interest in seeing the feature added. With some effort, it would seem that the performance cost for the disabled case could be reduced to an acceptable level, but if the main proponent is spending his time fighting and flaming, it seems unlikely that it will ever get merged. 

A newer set of patches, which just use `kzfree()` in specific sensitive places ([tty buffer management](<https://lwn.net/Articles/335945/>), [802.11 key handling](<https://lwn.net/Articles/335946/>), and [the crypto API](<https://lwn.net/Articles/335947/>)) were also proposed by Highsmith, but Linus Torvalds was not particularly [impressed](<https://lwn.net/Articles/335948/>). There was no need to use `kzfree()` there, a simple `memset()` was sufficient. Torvalds was not necessarily a believer in the need for the patches, nor for how Highsmith responded to review: 

but quite frankly, I'm not convinced about these patches at all. 

I'm also not in the least convinced about how you just dismiss everybodys concerns. 

There were some additional technical complaints about the patches as well, particularly the use of `kzfree()` _everywhere_ in the crypto API patch. Crypto API maintainer Herbert Xu [noted](<https://lwn.net/Articles/335951/>): ""The zeroing of metadata is gratuitous."" Overall, they had the look of being created grudgingly--as if it were a favor to do so. 

Where things go from here is unclear. Highsmith seemed to possibly be signing off in his [reply](<https://lwn.net/Articles/335954/>) to Torvalds: ""The next time a kernel vulnerability appears that is remotely related to some of the venues of attack I've commented, it will be useful to be able to refer to these responses."" There is some justification for Highsmith's frustration, but he needs to see that it isn't going to do him (or the kernel) any good. 

Kernel contributors, especially new ones, need to recognize that the community has folks that are at least as smart as they are. In this case, some of those developers may not have the security focus that Highsmith does, but that doesn't reduce their understanding of the kernel, nor their interest in seeing it have patches applied for better security. It would be unfortunate to see this feature, which could be very useful in some environments, fall by the wayside.
