---
title: "Faster reference-count overflow protection"
url: https://lwn.net/Articles/728675/
date: "July 24, 2017"
category: "Reference counting; Security-Kernel hardening"
author: "By Jonathan Corbet July 24, 2017"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 24, 2017

Improving the security of a system often involves tradeoffs, with the costs measured in terms of convenience and performance, among others. To their frustration, security-oriented developers often discover that the tolerance for these costs is quite low. Defenses against reference-count overflows have run into that sort of barrier, slowing their adoption considerably. Now, though, it would appear that a solution has been found to the performance cost imposed by reference-count hardening, clearing the way toward its adoption throughout the kernel. 

Reference-count overflows typically come about as the result of a programming error. Code that increments the reference count on an object may neglect to decrement it in certain error paths, for example. Such errors can allow an attacker to repeatedly increment a counter until it overflows, at which point the object in question can be made to appear to be unused and freed while it is, in fact, still in use. The resulting use-after-free vulnerability is often exploitable to fully compromise the system. 

The path toward protection against reference-count overflows in the kernel has been a long one. It started with code from the PaX/grsecurity patch set, but the initial approach of adding protection to the core `atomic_t` type [ran into opposition](<https://lwn.net/Articles/706498/>) and had to be changed. The next step was to introduce a new `refcount_t` type specifically for reference counts and to add the protections there. This type was [merged for the 4.11 development cycle](<https://lwn.net/Articles/715161/>) and various kernel subsystems were changed to use it, but `refcount_t` [upset the networking developers](<https://lwn.net/Articles/718275/>), who were unwilling to pay the performance cost associated with it. 

The networking layer is often where such patches run into trouble, but it was not the only place this time around. Andrew Morton recently [complained](<https://lwn.net/Articles/728680/>) about a `refcount_t` conversion in the interprocess communication (IPC) subsystem, for example, saying that there was no point in slowing down ""simple, safe, old, well-tested code"". It began to appear that, even if reference-count protection were added throughout the kernel, it would be disabled by distributors who feared the performance hit. 

One of the core truths of secure-systems development is that disabled (or never implemented) protective measures are remarkably ineffective at stopping attackers. Another one is that "safe, old, well-tested" code may be merely old, as Ingo Molnar [pointed out](<https://lwn.net/Articles/728681/>): 

It's old, well-tested code _for existing, sane parameters_, until someone finds a decade old bug in one of these with an insane parameters no-one stumbled upon so far, and builds an exploit on top of it. 

Truly protecting the kernel against reference-count overflows requires making the checks as universal as possible. That, in turn, requires either convincing developers to accept the performance cost of those checks or finding a way to reduce that cost to acceptable levels. The latter course is almost certainly the path of least resistance — if a solution to the performance cost can be found. 

**Update** : the single instruction mentioned to the left has been claimed by Pax Team as his work. The patch set remains Kees's. 

With his [fast refcount overflow protection patch set](<https://lwn.net/Articles/728626/>), Kees Cook would indeed appear to have found that solution. It works by adding a single instruction to the existing (highly optimized) `atomic_t` implementation that catches the case where the reference count goes negative (as happens when the counter overflows). The instruction is especially easy for the processor's branch-prediction logic to guess correctly, so it performs well, as demonstrated by microbenchmark results posted with the patch set. The standard `atomic_t` implementation ran the benchmark in 82.249 billion cycles; the new `refcount_t` code, instead, took 82.211 billion cycles — exactly the same within the margin of error, in other words. The older `refcount_t` implementation requires 144.8 billion cycles to run the test, for comparison. 

The current patch set is for the x86 architecture only. Since assembly work is required, each of the other architectures will need to be added individually when somebody gets around to doing it. There do not appear to be significant obstacles to making this technique work on the other major architectures. 

There is a cost to this change, relative to the full `refcount_t` implementation: it no longer detects the "increment from zero" case. If an object's reference count drops to zero, that object will normally be freed; a subsequent increment operation suggests that a reference still existed and the freed object may still be in use. This, obviously, would be a good situation to catch, but nobody has found a way to do so without adding to the expense of increment operations. Cook claimed in the patch set that the overflow case that the new `refcount_t` _does_ catch is the most common, though, and cited two exploits published in 2016 ([CVE-2014-2851](<http://cyseclabs.com/page?n=02012016>) and [CVE-2016-0728](<http://perception-point.io/2016/01/14/analysis-and-exploitation-of-a-linux-kernel-vulnerability-cve-2016-0728/>)) that would have been blocked had that checking been in place. 

There are still some developers who remain unenthusiastic about the `refcount_t` type; see [this complaint](<https://lwn.net/Articles/728684/>) from Eric Biederman (and [Cook's response](<https://lwn.net/Articles/728685/>)) for example. The remaining disagreements seemed to be based on a couple of arguments: (1) `refcount_t` doesn't fix all reference-count-related problems, and (2) using it implies a presumption of bugginess that some developers find hurtful to their pride. But, with the performance issue seemingly solved, those other complaints seem unlikely to block the implementation of reference-count hardening in most of the kernel. That can only be good news for those who are concerned about security.
