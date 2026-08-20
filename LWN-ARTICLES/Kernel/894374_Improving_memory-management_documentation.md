---
title: "Improving memory-management documentation"
url: https://lwn.net/Articles/894374/
date: "May 10, 2022"
category: "Documentation; Memory management-Documentation"
author: "By Jonathan Corbet May 10, 2022 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 10, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

Like much of the kernel, the memory-management subsystem is under-documented, and much of the documentation that does exist is less than fully current. At the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Mike Rapoport ran a session on memory-management documentation and what can be done to improve it. The result was a reinvigorated interest in documentation, but only time will tell what actual improvements will come from that interest. 

Rapoport started by noting that, a couple of years ago, he took a hard look at the current state of memory-management documentation. What he found was summarized as "Mel's book and some text files". The book in question, Mel Gorman's [_Understanding the Linux Virtual Memory Manager_](<https://www.kernel.org/doc/gorman/>), is reminiscent of many old Unix books: the basic concepts still apply to a great extent, but the details are all out of date and, thus, wrong. There are many important memory-management features, such as transparent huge pages, that are not mentioned at all. 

With regard to the text files in the kernel's documentation directory, Rapoport made the effort to convert them over to restructured text and integrate them into the kernel's documentation system, adding a bit of [![\[Mike Rapoport\]](https://static.lwn.net/images/conf/2022/lsfmm/MikeRapoport-sm.png)](<https://lwn.net/Articles/894376/>) much-needed organization in the process. He added some coverage of internal APIs, but there is a lot that is still in need of improvement. So, he asked, what can be done to improve the documentation and encourage the writing of more documentation? 

One idea, he continued, was for reviewers to make a point of reviewing the associated documentation when looking at memory-management patches. He has been making an effort in that direction, but has not seen other reviewers following suit. Matthew Wilcox jumped in to note that the [maple tree patches](<https://lwn.net/ml/linux-kernel/20220504010716.661115-1-Liam.Howlett@oracle.com/>) are well documented. Rapoport agreed, but said that doesn't change the fact that there is a lot of "tribal wisdom" in the memory-management community that does not exist in written form. 

Another developer noted that the documentation can be found in two distinct places: in the code, and under the kernel's documentation directory (`Documentation/`). The latter documentation, he said, is not as good as it could be. There are some sections written in a clear narrative file, but it is mixed in with "noise and horrible stuff". The rendered documentation, which incorporates kerneldoc comments from the code into the separate documents, can jumble everything together and can be hard to work with. Andrew Morton said that `Documentation/` is good for user-facing material, but otherwise the right place for documentation is in the code itself. 

As the maintainer for the documentation directory, I felt the need to jump in at this point; I had to disagree with Morton's assertion that separate documentation is only good for end users. There is a lot of information that is relevant to developers, but which doesn't fit readily into kerneldoc comments, and it is hard to tell a coherent story in the code that way. The idea that comments in the code will be better maintained than separate documentation is a poor match to reality at best. 

With regard to organization, it _is_ possible to put introductory and contextual information into kerneldoc comments and produce a coherent manual from them, but extra effort must be made toward that end, and the end result only appears in the documents after being rendered by the build system — not in the code. The [DRM documentation](<https://docs.kernel.org/gpu/index.html>) is a good example of what can be done when developers put effort into it. 

That said, organization has been an issue all along; when I became the documentation maintainer, the kernel's documentation directory was a seemingly random collection of independent files. Over the years, developers working on the documentation have been trying to organize that material with a focus on who the readers are; thus the [Core API manual](<https://docs.kernel.org/core-api/index.html>), the [User-space API guide](<https://docs.kernel.org/userspace-api/index.html>), the [Maintainer handbook](<https://docs.kernel.org/maintainer/index.html>), and several others. The net effect has been to create a set of smaller piles of unorganized and often outdated material, but it's a start. But people rarely find time to try to improve those manuals or to turn each into a coherent document rather than a collection of related files. 

Wilcox mentioned [Neil Brown's readahead documentation](<https://lwn.net/Articles/888715/>) as an example of another type of problem. The new documentation is "90% right"; Wilcox should have reviewed it but ~~was not copied on it~~ was unable to find the time. Brown, he said, did not use the documentation that was already present in the code when doing his work, and that is frustrating. 

A recurring theme was that there are not enough people with the time and expertise to work on documentation; developers were encouraged to lobby their employers to support that work. Michal Hocko said that you can't bring in a "random tech writer" to work on memory-management documentation, though; a lot of knowledge is needed to write useful documentation, so experienced people need to write it. Brown's approach was excellent, since he is an expert user of the interface and can see it through those eyes. Meanwhile, Hocko said, he generally avoids looking through the code when in search of documentation and digs through the LWN archives instead. 

I agreed with Hocko but had to add that writing documentation is a good way to gain the needed expertise. I learned much of what I know when working on [_Linux Device Drivers_](<https://lwn.net/Kernel/LDD3/>); it's fair to say that I was not well qualified when I began that project. 

Davidlohr Bueso claimed that the best document in the kernel, the one that others should emulate, is the infamous [`memory-barriers.txt`](<https://www.kernel.org/doc/Documentation/memory-barriers.txt>). It is written by developers with a high level of expertise, is clear, and actively maintained; even a newcomer can get something out of it. Johannes Weiner said that one of the strengths of `memory-barriers.txt` is that the document has had an excellent structure from the beginning; that made it relatively easy for others to come along and add to it. The memory-management subsystem needs somebody to come along and create a similar sort of documentation structure. 

Dan Williams asked what the near-term focus for memory-management documentation should be; did Rapoport have specific APIs in mind? Rapoport answered that his goal was to make the documentation better in general so that others could understand how Linux memory management works. Williams said that was "a good mission statement", but he was looking for actionable tasks. Rapoport suggested speculative page faults or the multi-generational LRU (both of which are still out of tree) as examples. 

Kent Overstreet said that developers are not bringing up documentation during code review, and that the subsystem does not have a person who has a coherent view of what the documentation should look like. Liam Howlett said that, as a new memory-management developer, he has encountered many functions that he did not understand. When he changes code, he tries to improve its documentation. He mentioned [`find_vma()`](<https://elixir.bootlin.com/linux/v5.18-rc6/source/mm/mmap.c#L2269>) specifically as a function whose behavior doesn't really match its name or documentation. 

David Hildenbrand asked what documents were wanted for memory management in general. Rapoport answered that he would like to see more material in the [admin guide](<https://docs.kernel.org/admin-guide/index.html>) first, preferably a high-level overview of how it all works. Improving the kerneldoc comments is rather lower on his list, but it is also easier to do. There was some discussion around whether there was a greater need for internal or user-oriented documentation; it was suggested that perhaps developers over-document some internal APIs, causing users to use them when they really should not. `find_vma()` was mentioned again as an example of this sort of problem. 

At the conclusion of the session, Wilcox suggested that a good first step would be to create a new memory-management document using Gorman's book as a guide, and volunteered to take a stab at it. That book had a structure that clearly worked; starting with that would solve the organizational problem and make it easy for developers to improve things. A ReStructured Text file could be created along those lines, and the existing documentation could be slotted into it as appropriate. There was a general agreement that this was a good thing to do — no doubt helped by the existence of a developer who was willing to take the initial steps. Wilcox has since posted [an initial version](<https://lwn.net/ml/linux-mm/20220503063032.3680308-1-willy@infradead.org/>) of the new documentation structure for review.
