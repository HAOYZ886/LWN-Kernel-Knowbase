---
title: A framework for page replacement policies
url: https://lwn.net/Articles/177118/
date: "March 25, 2006"
category: "CLOCK-Pro; Memory management-Page replacement algorithms"
author: 
---

""Holy cow."" 

That was Andrew Morton's [reaction](<https://lwn.net/Articles/177119/>) to a 34-part patch, posted by Peter Zijlstra, which creates an abstract API for page replacement policies. The page replacement code is at the core of the virtual memory system; it is, essentially, a set of heuristics which must decide which pages should be evicted from main memory and made available for other uses. Page replacement is a bit of a black art; it is easy to see when a system is managing memory poorly, but path to improvements is often far from clear. Memory management in Linux was a sore point for many years, but it seems to work well for most loads now. Given that all this tricky code has finally been beaten into reasonably good shape, why would anybody want to mess with it now? 

The answer is that there is quite a bit of research work going into alternative page replacement mechanisms, and Linux might just be able to benefit from some of that work. After all, few people would say that Linux virtual memory works so well that there is no room for improvement. 

This massive patch set creates an API for page replacement algorithms, allowing them to be changed at will. Or, at least, changed at reboot; there is currently no provision for loading replacement algorithms as modules or swapping them out on the fly. But, by selecting a page replacement scheme at kernel configuration time, system administrators can choose one which best suits their workload. Virtual memory hackers and others can play with different algorithms to see how they work out. And there is no need to pick one in particular as the page replacement algorithm for the Linux kernel. 

To work with this API, a page replacement algorithm must define a set of specific functions. Thus, for example, there is a pair of initialization functions: 

```
void page_replace_init(void);
        void page_replace_init_zone(struct zone *zone);
```

These functions, called at boot time, prepare the page replacement code to work with the system it finds itself running on. 

When the core kernel knows something about the use of specific pages, it can tell the replacement algorithm with these calls: 

```
void page_replace_hint_active(struct page *page);
        void page_replace_hint_use_once(struct page *page);
```

The first is called when the kernel notes that the page is in active use, while the second indicates that the page is unlikely to be used again in the near future. 

There are various other functions for helping with the housekeeping, but the core of the API is this function here: 

```
void page_replace_candidates(struct zone *zone, int count,
                                     struct list_head *list);
```

This function must select up to `count` pages from the given `zone` as candidates for eviction. This is where the page replacement code will gaze into its crystal ball to figure out which pages will not be used again anytime soon; those are the ones which will be singled out and passed back to the core kernel. 

Quite a few other functions exist. They deal with issues like page migration, tracking of non-resident pages, printing out information from the page replacement code, and more. See [the documentation file](<https://lwn.net/Articles/177123/>) for a full list and brief explanation of those other functions. 

The patch set also contains four different page replacement mechanisms. One is the modified least-recently-used (LRU) code found in current kernels, reworked to use the new API. Another is the CLOCK-PRO algorithm, [covered here](<http://lwn.net/Articles/147879/>) last August. There is an implementation of the CART technique, discussed in [this paper [PDF]](<http://www.almaden.ibm.com/cs/people/dmodha/clockfast.pdf>). Then there is a simple random replacement scheme, seemingly just for the fun of it. Actually, the [random replacement patch](<https://lwn.net/Articles/177127/>) is, due to its simplicity, a good place to start for somebody interested in seeing what a modularized page replacement algorithm looks like. 

This patch looks somewhat similar to the [pluggable CPU schedulers patch](<http://lwn.net/Articles/109458/>), which allows the scheduling algorithm to be changed. That patch continues to be maintained, but, since its initial posting in 2004, it has never been seriously considered for inclusion into the mainline kernel. There is a strong preference toward figuring out what's wrong - if anything - with the current code and fixing it, rather than creating a mechanism for playing with entirely different implementations. Thus, Andrew Morton followed his initial response with: 

Rather than replacing the whole lot four times I'd really prefer to see precise descriptions of these problems, see if we can improve the situation incrementally rather than wholesale slash-n-burn... 

Linus has [a similar opinion](<https://lwn.net/Articles/177124/>), and, additionally, is not convinced that page replacement is really an issue needing a great deal of attention. ""It smells like university research to me."" 

The proponents of this patch respond that there are, indeed, situations where the current code falls apart. Given that, the next logical step would seem to be gathering information on the cases where Linux memory management fails. Then the developers can start to think about what needs to be done to address those failures. Even if the page replacement framework patches are never merged, it looks like they may help to drive forward the next phase of work in Linux memory management algorithms. That should be a good thing regardless.
