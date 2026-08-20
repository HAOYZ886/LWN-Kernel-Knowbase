---
title: "The folio pull-request pushback"
url: https://lwn.net/Articles/868598/
date: "September 10, 2021"
category: "Memory management-Folios"
author: "By Jonathan Corbet September 10, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 10, 2021

When we last [caught up](<https://lwn.net/Articles/864603/>) with the [page folio](<https://lwn.net/Articles/849538/>) patch set, it appeared to be on track to be pulled into the mainline during the 5.15 merge window. Matthew Wilcox duly [sent a pull request](<https://lwn.net/ml/linux-kernel/YSPwmNNuuQhXNToQ@casper.infradead.org/>) in August to make that happen. While it is possible that folios could still end up in 5.15, that has not happened as of this writing and appears increasingly unlikely. What we got instead was a lengthy discussion on the merits of the folio approach. 

The kernel's memory-management subsystem deals with memory in pages, a fundamental unit that is generally set by the hardware (and is usually 4KB in size). These "base" pages are small, though, so the kernel often needs to deal with memory in larger units. To do so, it groups sets of physically contiguous pages together into compound pages, which are made up of a head page (the first base page of many in the compound page) and some number of tail pages. That leads to a situation where kernel code that is passed a `struct page` pointer often does not know if it is dealing with a head or a tail page without explicitly checking. 

It turns out that the "make sure this is a head page" checks add up to a certain amount of expense in a running kernel. The use of `struct page` everywhere also makes kernel APIs unclear — it can be difficult to know if a given function can cope with tail pages or not. To address this problem, Wilcox created the concept of a "folio", which is like a `struct page` but which is known not to be a tail page. By changing internal functions to use folios, Wilcox is able to speed up the kernel and clean up the API at the same time. The patch set is huge and intrusive, but it appeared to have overcome most resistance and be ready to head into the mainline kernel. 

#### Objections

Memory-management developer Johannes Weiner quickly [responded](<https://lwn.net/ml/linux-kernel/YSQSkSOWtJCE4g8p@cmpxchg.org/>) to the request to express his ""strong reservations"" about the folio concept. Over the course of the ensuing discussion he described his objections in a number of ways, but it seems to come down to a core point: a folio is just a collection of physically contiguous pages, and that is going to make it hard to deal with a number of challenges facing memory management. 

To start with, the folio design leaks too much information about memory-management internals to other users of folios, filesystems in particular. The current page-oriented APIs have the same problem, of course, but a massive API change should, he said, be the time to address that issue. So Weiner has [asked](<https://lwn.net/ml/linux-kernel/YSU7WCYAY+ZRy+Ke@cmpxchg.org/>), more than once, for the creation of a more abstract representation of memory that would be used in the higher levels of the kernel. This abstraction would hide a lot of details; it would also eliminate the assumption that a folio is a physically contiguous, power-of-two-sized chunk of memory. 

The assumption of physical contiguity, he [continued](<https://lwn.net/ml/linux-kernel/YSjxlNl9jeEX2Yff@cmpxchg.org/>), is a serious problem because the memory-management subsystem has never been good at allocating larger, contiguous chunks of memory. At some point fragmentation takes hold and those larger chunks simply aren't there. Techniques like page compaction can help to an extent, but that comes at the cost of excessive allocation latency. ""We've effectively declared bankruptcy on this already"", he said. There is no point in adopting folios to represent larger chunks of memory without thinking about how those chunks will be allocated. 

The other problem, he said, is that the folio concept could make it much harder to change the memory-management subsystem to use a larger base-page size. That change would make the system more efficient in many ways, including less memory wasted for `page` structures and less CPU time dedicated to dealing with them. There is one problem that has kept the kernel from increasing the base-page size for many years, though: internal fragmentation. 

When a file's contents (or a portion thereof) are stored in the page cache, they require a certain number of full pages. Unix-like systems have a lot of small files, but even a one-line file will occupy a full page in the page cache; all of the memory in that page beyond the end of the file is simply wasted. Increasing the size of a base page will necessarily increase the amount of memory lost to this internal fragmentation as well. In a previous folio discussion, Al Viro [did a quick calculation](<https://lwn.net/ml/linux-fsdevel/YGVUobKUMUtEy1PS@zeniv-ca.linux.org.uk/>) showing just how much more memory it would take to keep the kernel source in memory with a larger page size. A 64KB size would quadruple the memory used, for example; it is not a small cost. 

For this reason, Weiner [argued](<https://lwn.net/ml/linux-kernel/YS+7pzI7pttxtFHT@cmpxchg.org/>) that the kernel will need to be able to manage file caching in small units (the existing 4KB size, for example) even when the memory-management subsystem moves to a larger base-page size. In other words, the page cache will need to be able to work with sub-page units of memory. A new abstraction for memory might facilitate that; the current folio concept, being firmly tied to underlying pages, cannot. 

Wilcox's [answer](<https://lwn.net/ml/linux-kernel/YS1PzKLr2AWenbHF@casper.infradead.org/>) to this criticism seems to be that it makes little sense in managing memory in units other than the allocation size. The way to use larger units in the memory-management subsystem is thus to allocate compound pages, represented by folios; if everything in the kernel is using larger pages, memory will fragment less and those pages will become easier to allocate. For cases where smaller sizes are needed, such as page-cache entries for small files, simple base pages could be used. With careful allocation, the single pages could be packed together, further avoiding fragmentation. 

Weiner disagreed with that approach, though, saying that it puts the fragmentation-avoidance problem in the wrong place. The kernel has two levels of memory allocation: the page allocator (which deals in full pages and is where folios are relevant) and the slab allocator, which normally deals with smaller units. They have different strengths: 

> The page allocator is good at cranking out uniform, slightly big memory blocks. The slab allocator is good at subdividing those into smaller objects, neatly packed and grouped to facilitate contiguous reclaim, while providing detailed breakdowns of per-type memory usage and internal fragmentation to the user and to kernel developers. 

According to Weiner, Wilcox's approach forces the page allocator to deal with problems that are currently well solved in the slab allocator. ""As long as this is your ambition with the folio, I'm sorry but it's a NAK from me"". 

#### The _real_ problem with folios

The ultimate decision on the merging of folios is, of course, up to Linus Torvalds. Early in the conversation, he [wrote positively](<https://lwn.net/ml/linux-kernel/CAHk-=wgkA=RKJ-vke0EoOUK19Hv1f=47Da6pWAWQZPhjKD6WOg@mail.gmail.com/>) about the API improvements, but also noted that the patch set does bring a lot of churn to the memory-management subsystem. He concluded: ""So I don't hate the patches. I think they are clever, I think they are likely worthwhile, but I also certainly don't love them."" 

He also, however, noted that he wasn't entirely happy with the "folio" name, thus touching off one of the more predictable dynamics of kernel-community discussions: when the technical side gets involved and intractable, it must be time to have an extended debate on naming. So David Howells [suggested](<https://lwn.net/ml/linux-kernel/1957060.1629820467@warthog.procyon.org.uk/>) "sheaf" or "ream". Torvalds [favored](<https://lwn.net/ml/linux-kernel/CAHk-=wip=366HxkJvTfABuPUxwjGsFK4YYMgXNY9VSkJNp=-XA@mail.gmail.com/>) something more directly descriptive, like "head_page". Ted Ts'o [thought](<https://lwn.net/ml/linux-kernel/YSVH6k5plj9lrTFe@mit.edu/>) "mempages" would work, or [maybe](<https://lwn.net/ml/linux-kernel/YSVQOgrPhwGcUSp4@mit.edu/>) "pageset". Nicholas Piggin [favored](<https://lwn.net/ml/linux-kernel/1629948817.v8xwzejw2u.astroid@bobo.none/>) "cluster" or "superpage". Given the discussion, Vlastimil Babka [concluded](<https://lwn.net/ml/linux-kernel/793187d4-835f-a67e-392d-0d88e0a3a4fe@suse.cz/>) that the only suitable name was "pageshed". 

Wilcox has [made it abundantly clear](<https://lwn.net/ml/linux-kernel/YSVCAJDYShQke6Sy@casper.infradead.org/>) that he doesn't care about the name and will accept just about anything if that gets the code merged. He [redid the pull request](<https://lwn.net/ml/linux-kernel/YSmtjVTqR9%2F4W1aq@casper.infradead.org/>) with everything renamed to "pageset" just to prove that point. Needless to say, no real conclusion came from that branch of the conversation. 

At the end of August, Howells posted [a plea](<https://lwn.net/ml/linux-kernel/3285174.1630448147@warthog.procyon.org.uk/>) for a quick resolution on the issue; there is a lot of other pending memory-management work that either depends on the folio patches or conflicts with them. He asked: 

> Is it possible to take the folios patchset as-is and just live with the name, or just take Willy's rename-job (although it hasn't had linux-next soak time yet)? Or is the approach fundamentally flawed and in need of redoing? 

David Hildenbrand [added](<https://lwn.net/ml/linux-kernel/01ed765d-449d-fa5f-2f08-1b74e7f6a9c8@redhat.com/>) that he would like to see folios move out of linux-next one way or the other; sooner would be better. 

#### Now what?

For now, at least, the conversations have wound down. The 5.15 merge window is nearing its close and the folio patches have not been pulled. Chances are that means that folios will, at best, have to wait for another development cycle. That said, Torvalds has been known to hold controversial pulls until — or even past — the end of the merge window, when he has a bit more time to think them through. So even the closing of the merge window might not be an indication that the decision has been made. 

The final chapter has not been written here, but either way it seems clear that there is a lot of work yet to be done in the memory-management subsystem. Much of what needs to happen has yet to be designed, much less written and debugged; that adds some strength to one last [argument](<https://lwn.net/ml/linux-kernel/YSQeFPTMn5WpwyAa@casper.infradead.org/>) from Wilcox: ""The folio patch is here now"". Or, as Babka [asked](<https://lwn.net/ml/linux-kernel/6b01d707-3ead-015b-eb36-7e3870248a22@suse.cz/>): ""should we just do nothing until somebody turns that hypothetical future into code and we see whether it works or not?"" If folios go down in flames, work to improve memory-management internals at this level will have to restart from the beginning, and it's not clear that there is anybody out there who is ready to take up that challenge.
