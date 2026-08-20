---
title: "Explicit pinning of user-space pages"
url: https://lwn.net/Articles/807108/
date: "December 13, 2019"
category: "Memory management-get user pages"
author: "By Jonathan Corbet December 13, 2019"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 13, 2019

The saga of `get_user_pages()` — and the problems it causes within the kernel — has been extensively chronicled here; see [the LWN kernel index](<https://lwn.net/Kernel/Index/#Memory_management-get_user_pages>) for the full series. In short, `get_user_pages()` is used to pin user-space pages in memory for some sort of manipulation outside of the owning process(es); that manipulation can sometimes surprise other parts of the kernel that think they have exclusive rights to the pages in question. [This patch series](<https://lwn.net/ml/linux-kernel/20191211025318.457113-1-jhubbard@nvidia.com/>) from John Hubbard does not solve all of the problems, but it does create some infrastructure that may make a solution easier to come by. 

To simplify the situation somewhat, the problems with `get_user_pages()` come about in two ways. One of those happens when the kernel thinks that the contents of a page will not change, but some peripheral device writes new data there. The other arises with memory that is located on persistent-memory devices managed by a filesystem; pinning pages into memory deprives the filesystem of the ability to make layout changes involving those pages. The latter problem has been "solved" for now by disallowing long-lasting page pins on persistent-memory devices, but there are use cases calling for creating just that kind of pin, so better solutions are being sought. 

Part of the problem comes down to the fact that `get_user_pages()` does not perform any sort of special tracking of the pages it pins into RAM. It _does_ increment the reference count for each page, preventing it from being evicted from memory, but pages that have been pinned in this way are indistinguishable from pages that have acquired references in any of a vast number of other ways. So, while one can ask whether a page has references, it is not possible for kernel code to ask whether a page has been pinned for purposes like DMA I/O. 

Hubbard's patch set addresses the tracking part of the problem; it starts by introducing some new internal functions as alternatives to `get_user_pages()` and its variants: 

```
long pin_user_pages(unsigned long start, unsigned long nr_pages,
    		    	unsigned int gup_flags, struct page **pages,
    		    	struct vm_area_struct **vmas);
        long pin_user_pages_remote(struct task_struct *tsk, struct mm_struct *mm,
    			       unsigned long start, unsigned long nr_pages,
    			       unsigned int gup_flags, struct page **pages,
    			       struct vm_area_struct **vmas, int *locked);
        int pin_user_pages_fast(unsigned long start, int nr_pages,
    			    unsigned int gup_flags, struct page **pages);
```

From the caller's perspective, these new functions behave just like the `get_user_pages()` versions. Switching callers over is just a matter of changing the name of the function called. Pages pinned in this way must be released with the new `unpin_user_page()` and `unpin_user_pages()` functions; these are a replacement for `put_user_page()`, which was [introduced by Hubbard](<https://lwn.net/Articles/784574/>) earlier in 2019. 

The question of how a developer should choose between `get_user_pages()` and `pin_user_pages()` is somewhat addressed in the documentation update found in [this patch](<https://lwn.net/ml/linux-kernel/20191211025318.457113-11-jhubbard@nvidia.com/>). In short, if pages are being pinned for access to the data contained within those pages, `pin_user_pages()` should be used. For cases where the intent is to manipulate the `page` structures corresponding to the pages rather than the data within them, `get_user_pages()` is the correct interface. 

The new functions inform the kernel about the intent of the caller, but there is still the question of how pinned pages should be tracked. Some sort of reference count is required, since a given page might be pinned multiple times and must remain pinned until the last user has called `unpin_user_pages()`. The logical place for this reference count is in `struct page`, but there is a little problem: that structure is tightly packed with the information stored there now, and increasing its size is not an option. 

The solution that was chosen is to overload the page reference count. A call to `get_user_pages()` will increase that count by one, pinning it in place. A call to `pin_user_pages()`, instead, will increase the reference count by `GUP_PIN_COUNTING_BIAS`, which is defined in [patch 23 of the series](<https://lwn.net/ml/linux-kernel/20191211025318.457113-24-jhubbard@nvidia.com/>) as 1024. Kernel code can now check whether a page has been pinned in this way by calling `page_dma_pinned()`, which simply needs to check whether the reference count for the page in question is at least 1024. 

Using reference count in this way does cause a few little quirks. Should a page acquire 1024 or more ordinary references, it will now appear to be pinned for DMA. This behavior is acknowledged in the patch set, but is seen not to be a problem; false positives created in this way should not adversely affect the behavior of the system. A more potentially serious issue has to do with the fact that the reference count only has 21 bits of space; that means that only 11 bits are available for counting pins. That might be considered to be enough for most uses, but pinning a [compound page](<https://lwn.net/Articles/619514/>) causes the head page to be pinned once for each of the tail pages. A 1GB compound page contains 256 4KB pages, so such a page could only be pinned eight times before the reference count overflows. 

The solution to that problem, Hubbard says, is to teach `get_user_pages()` (and all the variants) about huge pages so that they can be managed with a single reference count. He notes that ""some work is required"" to implement this behavior, though, so it might not happen right away; it is certainly not a part of this patch set which, at 25 individual patches, is already large enough. 

There is one other little detail that isn't part of this set: how the kernel should actually respond to pages that have been pinned in this way. Or, as Hubbard puts it: ""What to do in response to encountering such a page is left to later patchsets"". One possibility can be found in [the layout lease proposal](<https://lwn.net/ml/linux-kernel/20190923190853.GA3781@iweiny-DESK2.sc.intel.com/>) from Ira Weiny, which would provide a mechanism by which long-term-pinned pages could be unpinned when the need arises. There is not yet a lot of agreement on how such a mechanism should work, though, so a full solution to the `get_user_pages()` problem is still a somewhat distant prospect. Expect it to be a topic for more heated discussion at the 2020 Linux Storage, Filesystem, and Memory-Management Summit. 

Meanwhile, though, the kernel may have at least gained a mechanism by which pinned pages can be recognized and tracked, which is a small step in the right direction. These patches have been through a number of revisions and look poised to enter Andrew Morton's -mm tree sometime in the near future. That would make their merging for the 5.6 kernel a relatively likely prospect.
