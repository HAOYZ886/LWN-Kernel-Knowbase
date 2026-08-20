---
title: Cleancache and Frontswap
url: https://lwn.net/Articles/386090/
date: "May 4, 2010"
category: "Memory management-Virtualization; Transcendent memory"
author: "By Jonathan Corbet May 4, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 4, 2010

Dan Magenheimer's [transcendent memory patch](<http://lwn.net/Articles/340080/>) was examined here last July. This patch creates a special class of memory which is not directly accessible to the rest of the kernel, allowing a number of special tricks to be played. Since then, transcendent memory has seemingly disappeared from view - until now, at least. Dan has returned with a pair of new abstractions - called "Cleancache" and "Frontswap" - each of which encapsulates a part of what transcendent memory does. 

[Cleancache](<http://lwn.net/Articles/384411/>) is the less controversial of the two. Dan describes it as ""a page-granularity victim cache for clean pages"", which should be crystal-clear to most LWN readers. For those who need a few more words: Cleancache provides a place where the kernel can put pages which it can afford to lose, but which it would like to keep around if possible. A classic example is file-backed pages which are clean, so they can be recovered from disk if need be. The kernel can drop such pages with no data loss, but things will get slower if the page is needed in the near future and must be read back from disk. 

In such situations, the kernel could, instead of dropping the page, put it into the Cleancache system with: 

```
int cleancache_put_page(struct page *page);
```

At some future point, if there is a need for the page, it can be retrieved with: 

```
int cleancache_get_page(struct page *page);
```

The key point is that there is never any guarantee that `cleancache_get_page()` will actually succeed in getting the page back. The Cleancache code (or whatever mechanism sits behind it) is free to drop the page at any time if it needs the memory for some other purpose. So Cleancache users must be prepared to fall back to the real backing store if `cleancache_get_page()` fails. 

While Cleancache holds the page, it can do creative things with it. Pages with duplicate contents are not uncommon, especially in virtualized situations; often, significant numbers of pages contain only zeroes. The backing store behind Cleancache can detect those duplicates and store a single copy. Compression of stored pages is also possible; there is currently work afoot to implement ramzswap ([CompCache](<http://lwn.net/Articles/334649/>)) as a Cleancache backend. It might also be possible to use Cleancache as part of a solid-state cache in front of a normal rotating drive. 

Dan's patches include the addition of hooks to commonly-used filesystems so that they will use Cleancache automatically. 

The other half of the equation is [Frontswap](<https://lwn.net/Articles/386103/>); unlike Cleancache, Frontswap is meant to deal with dirty pages that the kernel would like to get rid of. Once again, there is an interface for moving pages into and out of the system: 

```
int frontswap_put_page(struct page *page);
        int frontswap_get_page(struct page *page);
```

The rules are a bit different, though: Frontswap is not required to accept pages handed to it (so `frontswap_put_page()` can fail), but every page it accepts is guaranteed to be there later when the kernel asks to get it back. 

Like Cleancache, Frontswap can play tricks with the stored pages to stretch its memory resources. The real purpose behind this mechanism, though, appears to be to enable a hypervisor to respond quickly to memory usage spikes in virtualized guests. Dan [put it this way](<https://lwn.net/Articles/386104/>): 

Frontswap serves nicely as an emergency safety valve when a guest has given up (too) much of its memory via ballooning but unexpectedly has an urgent need that can't be serviced quickly enough by the balloon driver. 

Reviewers have been more skeptical of this mechanism. To some, it looks like a way for dealing with shortcomings in the balloon driver, which is already charged with implementing hypervisor decisions on how much memory is to be made available to guests. If that is the case, it seems like fixing the balloon driver might be the better approach. Dan's response is that balloon drivers cannot respond quickly to memory needs, and that regulating guest memory with a balloon driver can lead to swap storms. This is, apparently, a real problem encountered by virtualized systems in the field. If, instead, the hypervisor maintains a pool of pages for Frontswap, it can make them available quickly when the need arises, mitigating memory-related performance problems. 

Beyond that, Avi Kivity [complains](<https://lwn.net/Articles/386109/>) that memory given to guests with Frontswap can never be recovered by the hypervisor if those guests choose to hang onto it. Since operating systems tend to be written to take advantage of all of the memory resources available to them, it seems possible that Frontswap memory could fill quickly and would stay full, leaving the hypervisor starving for memory while maintaining pages it cannot get rid of. Avi also dislikes the page-at-a-time, synchronous nature of the Frontswap API. Dan's response here is that per-guest quotas will keep any guest from using too much Frontswap space and that the API is better suited to the problem being solved. 

Complaints notwithstanding, Cleancache and Frontswap already appear to be in reasonably wide use; they are shipping in OpenSUSE 11.2, Oracle's VM virtualization product, and with Xen. Such distribution certainly stretches the "upstream first" rule somewhat, but it also shows that there is apparently a real use case for these features. Given that the patches are not particularly intrusive and that the features have no cost if they are not used, it seems that something along these lines should make it into the mainline sooner or later.
