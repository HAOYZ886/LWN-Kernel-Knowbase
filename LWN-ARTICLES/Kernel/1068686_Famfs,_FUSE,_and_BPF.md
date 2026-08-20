---
title: "Famfs, FUSE, and BPF"
url: https://lwn.net/Articles/1068686/
date: "April 23, 2026"
category: "BPF-Filesystems; Filesystems-famfs"
author: "By Jonathan Corbet April 23, 2026"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 23, 2026

The famfs filesystem first [showed up on the mailing lists](<https://lwn.net/ml/all/cover.1708709155.git.john@groves.net/>) in early 2024; since then, it has been the topic of regular discussions at the Linux Storage, Filesystem, Memory Management and BPF (LSFMM+BPF) Summit. It has also, as result of those discussions, been through some significant changes since that initial posting. So it is not surprising that a suggestion that it needed to be rewritten yet again was not entirely well received. How much more rewriting will actually be needed is unclear, but more discussion appears certain. 

Famfs is designed to support large, read-mostly filesystems stored in shared memory. In practice, this means huge data sets kept in [CXL](<https://en.wikipedia.org/wiki/Compute_Express_Link>)-attached memory that is made available to multiple systems simultaneously. In normal usage, software running on those systems will access this data by mapping it directly into its address space with [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>), so that the data is all immediately accessible without system calls, and without going through the system's page cache. It is possible to perform normal filesystem reads and writes, though write access is only minimally supported. 

In its initial form, famfs was implemented like any other standalone filesystem, but it included a user-space component that drew a fair amount of attention from the filesystem developers at [the 2024 LSFMM+BPF session](<https://lwn.net/Articles/983105/>). Given that some of famfs is already implemented in user space, they suggested, it might be better to just use [FUSE](<https://docs.kernel.org/filesystems/fuse/>), which was designed for just that kind of filesystem. At the time, famfs creator John Groves said that he was unsure about whether FUSE would work, but would be willing to give it a try. His concern was that famfs must operate at memory speeds, and could thus not afford to call into user space to resolve page faults. 

At the 2025 LSFMM+BPF gathering, Groves [returned](<https://lwn.net/Articles/1020170/>) with a shiny new FUSE-based implementation that appeared to have solved that problem. It introduces two new FUSE operations toward that goal. `GET_FMAP` is invoked when a file is opened; the user-space server responds by providing the kernel with a list of memory locations and lengths, providing a map of how the file is laid out in shared memory. Thereafter, the kernel is able to resolve page faults without having to go back to user space for more information. The other operation, `GET_DAXDEV`, provides information about the CXL devices on which the shared memory is hosted. 

There did not appear to be fundamental objections to the FUSE implementation. Over the following year, Groves worked on refining the code; when he posted [version 10](<https://lwn.net/ml/all/0100019d43e5f632-f5862a3e-361c-4b54-a9a6-96c242a8f17a-000000@email.amazonses.com>) at the end of March, he had every reason to believe that the work, which he had [described](<https://lwn.net/ml/linux-fsdevel/0100019c2bdca8b7-b1760667-a4e6-4a52-b976-9f039e65b976-000000@email.amazonses.com/>) in February as having been ""kinda hard"", was close to ready for merging into the mainline. But, as he discovered, merging is never certain until it actually happens. 

#### Better done in BPF?

Joanne Koong, who has done a fair amount of work with FUSE over the years, [asked](<https://lwn.net/ml/all/CAJnrk1ZRTGWjNzkMxS3UkeZMmrpadJDtWKontMx2=d-smXYq=w@mail.gmail.com>) a seemingly simple question: 

> I'm curious to hear your thoughts on whether you think it makes sense for the famfs-specific logic in this series to be moved to a bpf program that goes through a generic fuse iomap dax layer. 

Changing the code in that way, Koong suggested, would make the FUSE logic more extensible and applicable to other types of filesystems. It would also bring more flexibility to famfs, making it easier, for example, to adjust to changes to files after they have been opened, and allowing famfs updates to be made available more quickly to users, since they would not have to wait for the usual kernel release cycle. She also [said](<https://lwn.net/ml/all/CAJnrk1a06zkUmXW5EFiUmgAoFauwtzsYvnotaPH0ifVtyh7iDQ@mail.gmail.com>) that she had posted a prototype implementation of a BPF-based famfs back in November, and suggested that switching over to this approach would not involve a huge rewrite of the famfs code. 

The FUSE maintainer is Miklos Szeredi; he [entered the conversation](<https://lwn.net/ml/all/CAJfpegvVTcV89=q3L326aGQjhduBcv7PVg5QKftGLjNZmCLmaw@mail.gmail.com>) saying that he would prefer to avoid adding a famfs-specific FUSE interface if it could be avoided. The BPF idea thus appealed to him; he suggested that it should be given a try before considering a merge of the existing famfs patches. 

It is fair to say (and understandable) that Groves was not entirely pleased by this turn in the conversation. He would, he [said](<https://lwn.net/ml/all/adkDq0m5Wt9YhJ8A@groves.net>), ""object vehemently"" to being required to undertake this rewrite before the code could be merged. The current implementation, he said, matches what had been asked of him two years ago. He later [added](<https://lwn.net/ml/all/ad4_jFsR951c2Mtn@groves.net>) that there would be some real risks involved with the BPF approach, starting with the fact that he would have to learn how to work with BPF, and the performance impacts of such a change would be unknown. The current version is already shipping to users, he said; it is too late to demand such changes now. 

#### Possible solutions

The purpose of the famfs user-space component is to determine where the extents of a given file have been placed in memory and to inform the kernel of those placements. The BPF alternative would work similarly, with user space providing that information in a filesystem-independent way; the BPF program would then provide the filesystem-specific interpretation for the rest of the kernel. One possibility, for example, could be for user space to store the extent information in either a BPF map or an [arena](<https://lwn.net/Articles/961941/>). 

Various attempts at a solution along these lines exist already. As Darrick Wong [pointed out](<https://lwn.net/ml/all/20260414185740.GA604658@frogsfrogsfrogs>) in the discussion, he had [posted one such](<https://lwn.net/ml/linux-fsdevel/177188736765.3938194.6770791688236041940.stgit@frogsfrogsfrogs/>) in February, based loosely on Koong's work. In short, it provides access to the kernel's [iomap](<https://docs.kernel.org/filesystems/iomap/>) layer, with BPF hooks to help FUSE filesystems complete the mappings. Wong complained that he had not gotten any review responses, and expressed disappointment with the pace of reviews in the FUSE subsystem in general. He [thought](<https://lwn.net/ml/all/20260414233631.GB604658@frogsfrogsfrogs>) that a BPF-based implementation could be upstreamed within two development cycles — if the relevant maintainers would accept it. Whether that would happen, he suggested, is far from clear: 

> The issues I was alluding to are BPF being used as a means to get around slow/unresponsive maintainers; and the kernel community's collective refusal to explore any other path to building new user APIs besides designing everything generically perfectly up front in the kernel UABI along with all the stress that involves. 

Rather than trying to develop the perfect API from the beginning, he [later said](<https://lwn.net/ml/all/20260416224331.GD114184@frogsfrogsfrogs>), the best approach might be to merge famfs in its current form, then experiment with alternative approaches afterward. If the interface is carefully designed, it should be possible to move to a better one in the future, should one be found. Others in the conversation also suggested that this might be the best way. Christoph Hellwig, though, [was strongly against that idea](<https://lwn.net/ml/all/aeHrsIGagdmZJ1Fw@infradead.org>), saying that the multitude of approaches under consideration showed that more thought needed to be put into designing a single interface. 

Gregory Price, meanwhile, [complained](<https://lwn.net/ml/all/ad69tTnx5YkD4Y9K@gourry-fedora-PF4VCD3F>) that working software was being held back in favor of an unproven approach that might well offer worse performance; ""John is right to push back here"". But he also suggested that the existing interface might, in fact, be more generic than it appears, and could be the basis for a longer-term solution as well: 

> That said - I'm looking at fs/fuse/famfs.c and I'm asking myself what in here is actually famfs-specific. If you just s/FAMFS/DAX/g \- the file just reads like a simple DAX-iomap backend with optional striping. 
> 
> Would it be reasonable to refactor the dax layer (and users) to create an ops structure that becomes the basis for the BPF solution? 

That led memory-management developer David Hildenbrand to [ask](<https://lwn.net/ml/all/f254f6fc-dc06-4612-82d7-35bb10dbd32e@kernel.org>) whether the BPF solution would be acceptable to the memory-management developers, a question that Groves [was also acutely interested in](<https://lwn.net/ml/all/aeUU8hMwPij2WvfF@groves.net>). If the answer is "no", Groves said, much of the discussion described here would be moot. Meanwhile, he added, he had just received a prototype implementation from Price that could be interesting; Price then [described that solution](<https://lwn.net/ml/all/aeVy2MzucnrLlOQx@gourry-fedora-PF4VCD3F>), which involves a BPF callback at file-open time to do the equivalent of the `GET_FMAP` call. 

As of this writing, Groves is evaluating how well Price's prototype implementation will work. It seems clear, though, that no conclusion will be reached in the email discussion. The next LSFMM+BPF meeting, as it happens, is the first week in May. That will be the perfect opportunity to lock the filesystem, memory-management, and BPF developers into the same room and deprive them of beer until they come up with a solution that all can live with.
