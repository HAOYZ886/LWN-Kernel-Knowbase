---
title: "Intel's new 64-bit architecture and Linux"
url: https://lwn.net/Articles/72905/
date: "February 25, 2004"
category: ia32e
author: 
---

Intel has run into a problem that, sooner or later, catches up with many major vendors in the computing industry: customers like standard technologies. It is difficult to introduce a product which ignores the prevailing standards - even if you are the company which set those standards in the first place. Thus, the "Intel" name has not been enough to push the industry toward its Itanium processors. Instead, vendors have been incorporating AMD's 64-bit processors, which retain x86 compatibility and extend that architecture in a relatively natural way. 

In response, Intel has finally [unveiled](<http://www.intel.com/technology/64bitextensions/index.htm?iid=techtrends+spotlight_64bit>) its own 64-bit extensions, under the "ia32e" name. Intel itself does not say this, but a review of the new architecture revealed fairly quickly that Intel has adopted (for the most part) AMD's 64-bit architecture. Intel is now in the business of selling AMD-compatible processors. Linus was [rather annoyed](<https://lwn.net/Articles/72907/>) at Intel for not coming out and just saying this, to the point that he toyed with the idea of renaming the kernel's x86-64 architecture "AMD64." Calm thinking prevailed, however, and Linus [chose](<https://lwn.net/Articles/72909/>) to stick with a vendor-neutral name. 

Support for the new architecture has already been merged into the (upcoming) 2.6.4 kernel; the [patch](<https://lwn.net/Articles/72904/>) came from Andi Kleen. Given the great similarities with the AMD64 architecture, this support was relatively easy to implement. Intel may not have been entirely straightforward about the path it has taken, but, where it matters, Intel has done the right thing.
