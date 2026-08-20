---
title: "Readahead: the documentation I wanted to read"
url: https://lwn.net/Articles/888715/
date: "April 8, 2022"
category: "Memory management-Documentation; Memory management-Readahead; Readahead"
author: "April 8, 2022 This article was contributed by Neil Brown"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

April 8, 2022

This article was contributed by Neil Brown

The readahead code in the Linux kernel is nominally responsible for reading data that has not yet been explicitly requested from storage, with the idea that it might be needed soon. The code is stable, functional, widely used, and uncontroversial, so it is reasonable to expect the code to be of high quality, and largely this is true. Recently, I found the need to document this code, which naturally shone a rather different light on it. This work revealed minor problems with functionality and significant problems with naming. 

My particular reason for wanting documentation probably colors my view of the code so I'll start there. Once upon a time, Linux had a strong concept of "congestion" as it applied to I/O paths. If the queue of requests to some device grew too large, the backing device would be marked as "congested" and certain optional I/O requests would be skipped or delayed, particularly writeback and readahead. As time has passed, [so too (apparently) has the need for congestion management](<https://lwn.net/Articles/873672/>). Maybe this is because many I/O devices are now faster than our CPUs but, whatever the reason, the block layer no longer tracks congestion and only a few virtual "backing devices" continue this outdated practice. 

In Linux 5.16, the only backing device that gets marked as "read congested" is the virtual device used for FUSE filesystems. As part of a project to remove all remnants of congestion tracking, I proposed that there was really nothing special about FUSE, and it should just accept all readahead requests just like everyone else. Miklos Szeredi, the maintainer of FUSE, [found my reasoning to be unsatisfactory](<https://lwn.net/ml/linux-kernel/CAJfpegt-igF8HqsDUcMzfU0jYv8WpofLy0Uv0YnXLzsfx=tkGg@mail.gmail.com/>) — and who could blame him? If FUSE doesn't want readahead requests, it shouldn't have to accept them. Trying to understand how FUSE could safely say "no" to readahead, without having to maintain the congestion-tracking functionality in common code, started me on the path to understanding readahead — once it was [explained to me](<https://lwn.net/ml/linux-kernel/YfiUaJ59A3px+DqP@casper.infradead.org/>) that it wasn't as simple as just changing the "readahead" callback in FUSE to return zero. 

The main part of the API exported by `mm/readahead.c` is two functions: [`page_cache_sync_ra()`](<https://elixir.bootlin.com/linux/v5.17/source/mm/readahead.c#L553>) and [`page_cache_async_ra()`](<https://elixir.bootlin.com/linux/v5.17/source/mm/readahead.c#L583>). This functionality is also available with a slightly simpler interface as [`page_cache_sync_readahead()`](<https://elixir.bootlin.com/linux/v5.17/source/include/linux/pagemap.h#L1037>) and [`page_cache_async_readahead()`](<https://elixir.bootlin.com/linux/v5.17/source/include/linux/pagemap.h#L1058>), which are nicely documented in [the kernel documentation](<https://www.kernel.org/doc/html/v5.18-rc1/core-api/mm-api.html?highlight=page_cache_async_readahead#c.page_cache_sync_readahead>). 

#### Sync and async

Unfortunately, that documentation is not explicit on how the "sync" or "async" in the names are relevant. Clarifying this was among my first tasks so, to help with that clarification, I'll refer you to a selection from [my new documentation](<https://www.kernel.org/doc/html/v5.18-rc1/core-api/mm-api.html#readahead>), which was [merged for the 5.18 release](<https://git.kernel.org/linus/84dacdbd5352>). It starts: 

> Readahead is used to read content into the page cache before it is explicitly requested by the application. Readahead only ever attempts to read pages that are not yet in the page cache. If a page is present but not up-to-date, readahead will not try to read it. In that case a simple ->readpage() will be requested. 
> 
> Readahead is triggered when an application read request (whether a system call or a page fault) finds that the requested page is not in the page cache, or that it is in the page cache and has the `PG_readahead` flag set. This flag indicates that the page was loaded as part of a previous readahead request and now that it has been accessed, it is time for the next read-ahead. 
> 
> Each readahead request is partly synchronous read, and partly async readahead. 

We stop here, in mid-paragraph, to focus on those two terms: sync and async. Readahead is, by its nature, asynchronous — nothing is waiting for it. An explicitly requested read, instead, will ultimately be synchronous, as the operation cannot complete until the data arrives. These two modes are clearly related and handling them both in the same code makes sense. Describing them both as being "readahead" — a choice that was effectively forced on me by the code — is not so defensible. 

Anyone who has been around computers long enough to know that a "kilobyte" isn't (necessarily) 1000 bytes will also know that we technologists often follow the practice of Lewis Carroll's "Humpty Dumpty" in [_Through the Looking Glass_](<https://www.gutenberg.org/files/12/12-h/12-h.htm>): 

> "When I use a word," Humpty Dumpty said in rather a scornful tone, "it means just what I choose it to mean — neither more nor less." 

We seem to make that mistake rather more than is good for us, and the readahead code is certainly not innocent. 

Each filesystem can provide an [`address_space_operations`](<https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L363>) method, named `readahead()`, to initiate a read; it is on this basis that the term "readahead request" is used in the documentation. There is also an address-space operation called `readpages()`, though it was [marked as deprecated](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8151b4c8bee43cea7a28cb0300123df90880e60c>) in the middle of 2020 and will be [removed for 5.18](<https://git.kernel.org/linus/704528d895dd>). These two functions have much the same functionality (they both issue read requests for a collection of pages). The newer `readahead()` has a much better interface (the details are beyond the scope of this article), but `readpages()` has undoubtedly the better name — because that is what they both do. They don't just "read ahead" but also issue reads that have explicitly been requested. 

Once one realizes that the functionality of `readahead()` is just to submit read requests, some of which the caller will wait for ("sync") and some of which the caller won't wait for ("async"), the intention of the code starts to become a lot clearer. Names matter. 

#### When readahead can be skipped

Returning to the original problem of giving FUSE the opportunity to skip readahead, a way forward now appears. The `readahead()` function that FUSE supplies must read all the pages that will be waited for, but it doesn't need to read the remainder. One of the improvements to the interface that came with the introduction of the `readahead()` operation is that more information is available to the filesystem. This information includes a `struct file_ra_state`, which contains a field called `async_size`. Aha! This must be the size of the readahead section. 

Or is it? Can we trust the name? This structure is, fortunately, [documented](<https://www.kernel.org/doc/html/v5.18-rc1/filesystems/api-summary.html?highlight=file_ra_state#c.file_ra_state>); the description for this `async_size` field reads: ""Start next readahead when this many pages are left"". What does that mean, and what does it have to do with being "async"? Possibly reading some more of the new documentation will help. 

> Each readahead request is partly synchronous read, and partly async readahead. This is reflected in the struct file_ra_state which contains ->size being to total number of pages, and ->async_size which is the number of pages in the async section. The first page in this async section will have PG_readahead set as a trigger for a subsequent readahead. Once a series of sequential reads has been established, there should be no need for a synchronous component and all readahead requests will be fully asynchronous. 

The second sentence, which presents the meaning of `async_size`, is something I made up — it was not previously present in any documentation and is not completely consistent with the code, though it matches the field name perfectly. The third sentence, about the `PG_readahead` flag, matches the code and pre-existing documentation. 

A core idea in readahead is to take a risk and read more than was requested. If that risk brings rewards and the extra data is accessed, then that justifies a further risk of reading even more data that hasn't been requested. When performing a single sequential read through a file, the details of past behavior can easily be stored in the `struct file_ra_state`. However if an application reads from two, three, or more, sections of the file and interleaves these sequential reads, then `file_ra_state` cannot keep track of all that state. Instead we rely on the content already in the page cache. Specifically we have a flag, `PG_readahead`, which can be set on a page. That name should be read in the past tense: the page was read ahead. A risk was taken when reading that page so, if it pays off and the page is accessed, then that is justification for taking another risk and reading some more. 

Which page should this flag be set on? Another core premise of readahead is that reads are often sequential, and it is only on this basis that we take a risk and read the following pages. So if the first of the ahead-read pages is accessed, then a sequential read can be assumed. If some later page is read, less can be concluded. It seems clear to me that `PG_readahead` must be set on the first of the pages that were opportunistically requested. This is consistent with the documented behavior of setting it based on the value of `async_size`, and is consistent with _most_ of the code, though there are a couple of places where some different value is used with no clear justification. 

This, then, is enough to allow FUSE to choose when to skip pages in its `readahead()` handler — it looks at the `async_size` value — but it isn't quite enough for completely correct behavior. When `readahead()` is called, the pages of memory have already been added to the page cache, though they have not yet been marked up-to-date. Leaving them there without initiating a read can result in later attempts to read them being less efficient. This can [easily be fixed](<https://www.spinics.net/lists/mm-commits/msg163315.html>) by having the caller drop pages from the page cache if the `readahead()` function chose to ignore them and indicated this by _not_ updating some (private) fields in [`struct readahead_control`](<https://elixir.bootlin.com/linux/v5.17/source/include/linux/pagemap.h#L993>). 

#### Oddities

There is a bit more to the documentation, and there are more oddities that only came to light because of the need to document. So, picking up where we left off: 

> When either of the triggers causes a readahead, three numbers need to be determined: the start of the region, the size of the region, and the size of the async tail. 
> 
> The start of the region is simply the first page address at or after the accessed address, which is not currently populated in the page cache. This is found with a simple search in the page cache. 
> 
> The size of the async tail is determined by subtracting the size that was explicitly requested from the determined request size, unless this would be less than zero — then zero is used. NOTE THIS CALCULATION IS WRONG WHEN THE START OF THE REGION IS NOT THE ACCESSED PAGE. 

Often I have used the act of documentation as a means for finding and fixing bugs — if accurate documentation starts to look contorted, it can be easier to fix the code first so as to allow the documentation to be more coherent. In this case I chose to leave the document clumsy, in part because naming was again a problem, and as we know, finding good names is hard. 

As mentioned, the readahead code has two API functions with unfortunate names: `page_cache_sync_ra()` and `page_cache_async_ra()`. These are called in response to the two triggers — when trying to access a page that is not cached, or when accessing a page that was flagged as `PG_readahead`. Both might issue reads that will soon be waited on (sync) as well as reads that might not be (async). 

Each of these functions has a final argument called `req_count`, which is the count of pages in the initial request. The implication is that we need at least that many pages, but it is OK to request more if that seems appropriate. It is the meaning and use of `req_count` that resulted in that loud NOTE ending that above section of documentation. 

Interpreting `req_count` as "size of the initial request" matches the name, but it isn't always obvious that this is the number being passed in. As we have seen, the logic in the readahead code is mostly about guessing how much data might be needed in the future. Some callers of these functions already _know_ that a sequential read is happening, for example because the [`madvise()` system call](<https://man7.org/linux/man-pages/man2/madvise.2.html>) has been used to declare the application's intentions. In these cases, `req_count` is set to a suitably large number. This isn't exactly the number of pages that are needed now, but it is the number of pages that are _known_ to be wanted, so these are pages that the filesystem should not skip just because it is inconvenient to read them just now. 

With the caveat that the request may include explicitly requested future pages, it is fairly clear what `req_count` means, but how is it used? Before diving in to explore this, it will help to read a bit more of the new documentation to understand how the size of a readahead request is calculated. 

> The size of the region is normally determined from the size of the previous readahead which loaded the preceding pages. This may be discovered from the struct file_ra_state for simple sequential reads, or from examining the state of the page cache when multiple sequential reads are interleaved. Specifically: where the readahead was triggered by the `PG_readahead` flag, the size of the previous readahead is assumed to be the number of pages from the triggering page to the start of the new readahead. In these cases, the size of the previous readahead is scaled, often doubled, for the new readahead, though see get_next_ra_size() for details. 

For the `page_cache_sync_ra()` case, called when a wanted page is missing, one would expect `req_count` to be at least one, and that is in fact the case. Some number of pages will be allocated, depending how big a hole there is in the page cache, how big the request is, and how much readahead seems justified. These pages are added to the page cache and the filesystem's `readahead()` function is called to load them. 

When `page_cache_async_ra()` is called because a `PG_readahead` flagged page was found, the situation is different. The set of pages that will be read will _not_ include the page that was just found (it has already been read) and probably not some number of subsequent pages. The code will search through the page cache for the first missing page, and consider reading from there. How many of these pages will be among those needed for the initial request? Maybe some, certainly not `req_count` of them. 

That last claim of the relationship between `req_count` and the pages actually read is based on assumptions which, as I have occasionally suggested, are not always completely consistent with the code. To be sure, we need to go back to the code and see how `req_count` is actually used in the `page_cache_async_ra()` case. Fortunately we have many years of development history in Git, and the documentation found for individual patches is often better than documentation found in the code. 

#### `req_count` through the ages

Prior to Linux 2.6.31, `req_count` (then called `req_size`) wasn't used for the reads triggered by `PG_readahead` at all. The [change](<https://git.kernel.org/linus/160334a0cfa8>) in that release caused it to be used to increase the size of ahead reads. Previously, this was calculated as the size of the previous ahead read, scaled up by a factor of two or four. Since then, it is the size of the previous ahead read _plus` req_count`_, and then scaled up. The justification for this change was: 

> Make sure interleaved readahead size is larger than request size. This also makes the readahead window grow up more quickly. 

Unfortunately, there is no indication of the sort of workload that would benefit from this change. To me, it has the appearance of `req_count` being used not because it was the right number based on some theoretical analysis, but because it was an easily available number that was about the right size. So this doesn't provide much insight into what `req_count` is supposed to _mean_. 

Then, in Linux 4.10, `req_count` found a new use. [That patch](<https://git.kernel.org/linus/9491ae4aade6>) allowed the number of pages requested in the readahead process to be at least the size of the original request, even if that is larger that maximum readahead size that is configured (as long as it wasn't bigger than the device was configured to accept). This is a clear acknowledgment that part of the "readahead" is really a synchronous read, not to be constrained by readahead limits. It also emphasizes that `req_count` isn't simply a size (maybe to be used for scaling), but it identifies a specific set of pages — from the starting point of the request. So when the starting point for readahead is moved forward over any pages that are already in the page cache, the `req_count` really should be reduced by the number of pages skipped over. Only then will it still signify the number of pages that are part of the original request, which still need to be read and which can justify exceeding the maximum readahead size. 

From a purely behavioral perspective, this lack of clarity over the meaning of this parameter may not be all that important. Readahead size calculations are heuristics. There is no right answer and, if a couple of extra fudge factors slip in by mistake, it is just a different heuristic. But from the perspective of wanting to understand the code, and particularly of wanting to change the code without breaking anything, this sort of detail can be quite important. 

As mentioned, I want the filesystem to know how many pages were explicitly requested, and how many were heuristically suggested. This requires a clear understanding of what `req_count` means. Getting slightly incorrect data may not hurt a lot, but it certainly doesn't help. 

#### The rest of the story

And now we can read the remainder of the documentation, which hopefully will integrate some of the ideas already explored. As it is aimed at people who are already generally familiar with the Linux page cache, it contains some concepts such as page locking that are best just skipped over by the casual reader. 

> If the size of the previous read cannot be determined, the number of preceding pages in the page cache is used to estimate the size of a previous read. This estimate could easily be misled by random reads being coincidentally adjacent, so it is ignored unless it is larger than the current request, and it is not scaled up, unless it is at the start of file. 
> 
> In general, readahead is accelerated at the start of the file, as reads from there are often sequential. There are other minor adjustments to the readahead size in various special cases and these are best discovered by reading the code. 
> 
> The above calculation, based on the previous readahead size, determines the size of the readahead operation, to which any requested read size may be added. 
> 
> Readahead requests are sent to the filesystem using the ->readahead() address space operation, for which mpage_readahead() is a canonical implementation. ->readahead() should normally initiate reads on all pages, but may fail to read any or all pages without causing an I/O error. The page cache reading code will issue a ->readpage() request for any page which ->readahead() does not provide, and only an error from this will be final. 
> 
> ->readahead() will generally call readahead_page() repeatedly to get each page from those prepared for readahead. It may fail to read a page by: 
> 
>   * not calling readahead_page() sufficiently many times, effectively ignoring some pages, as might be appropriate if the path to storage is congested. 
>   * failing to actually submit a read request for a given page, possibly due to insufficient resources, or 
>   * getting an error during subsequent processing of a request. 
> 

> 
> In the last two cases, the page should be unlocked to indicate that the read attempt has failed. In the first case the page will be unlocked by the caller. 
> 
> Those pages not in the final `async_size` of the request should be considered to be important and ->readahead() should not fail them due to congestion or temporary resource unavailability, but should wait for necessary resources (e.g. memory or indexing information) to become available. Pages in the final `async_size` may be considered less urgent and failure to read them is more acceptable. In this case it is best to use delete_from_page_cache() to remove the pages from the page cache as is automatically done for pages that were not fetched with readahead_page(). This will allow a subsequent synchronous readahead request to try them again. If they are left in the page cache, then they will be read individually using ->readpage(). 

The purpose of writing the documentation was to ensure that I understood the code and ensure that others would be able to understand my motivation for changes to that code. It has, I think, achieved that. However it has also opened up opportunities for making the code, and the names used in the code, more transparent. While I would like such improvements to happen, I'm not sure when I'll find time — through I would make time to help if someone else wanted to drive the effort. 

The purpose of this meta-narrative about the writing of the documentation is different. I wanted to highlight the difficulty of maintaining a coherent "intent" or "meaning" of various details of the code, as it is modified by various people at various times. Meanings can drift, inconsistencies can accumulate, misnomers can become entrenched. As a community we have found that the best way to maximize correctness and consistency is to have tools that alert us to problems. Until we have tools that can read documentation (including the implicit documentation of variable names) and highlight inconsistencies, that is one part of the process that we will have to continue doing ourselves. 

So please: when you change code, also change the documentation. And if there isn't any documentation yet — write some!
