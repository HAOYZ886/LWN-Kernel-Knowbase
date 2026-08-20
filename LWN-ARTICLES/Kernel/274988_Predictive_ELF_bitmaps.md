---
title: Predictive ELF bitmaps
url: https://lwn.net/Articles/274988/
date: "March 26, 2008"
category: Prefetching
author: "By Jake Edge March 26, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
March 26, 2008

When the kernel executes a program, it must retrieve the code from disk, which it normally does by demand paging it in as required by the execution path. If the kernel could somehow know which pages would be needed, it could page them in more efficiently. Andi Kleen has posted an experimental [set of patches](<http://lwn.net/Articles/273800/>) that do just that. 

Programs do not know about their layout on disk, nor is their path through the executable file optimized to reduce seeking, but with some information about which pages will be needed, the kernel _can_ optimize the disk accesses. If one were to gather a list of the pages that get faulted in as a program runs, that information could be saved for future runs. It could then be turned into a bitmap indicating which of the pages should be prefetched. 

Once you have such a bitmap, where to store it becomes a problem. Kleen's method uses a "hack" to the ELF format on disk, putting the bitmap at the end of the executable. This has a number of drawbacks: a seek to get the info, modifying the executable each time you train, and only allowing a single usage pattern system-wide. It does have one very nice attribute, though, the bitmap and executable stay in sync; if the executable changes, due to an upgrade for instance, the bitmap would get cleared in the process. Alternative bitmap storage locations--somewhere in users' home directories for example--do not have this property. 

Andrew Morton [questions whether this need be done in the kernel at all](<http://lwn.net/Articles/275053/>): 

Can't this all be done in userspace? Hook into exit() with an LD_PRELOAD, use /proc/self/maps and the new pagemap code to work out which pages of which files were faulted in, write that info into the elf file (or a separate per-executable shadow file), then use that info the next time the app is executed, either with an LD_PRELOAD or just a wrapper. 

Ulrich Drepper does not want to see the ELF format abused in the fashion it was for this patch, Kleen doesn't either, but used it as an expedient. Drepper thinks the linker should be taught to emit a new header type which would store the bitmap. It would be near the beginning of the ELF file, eliminating the seek. A problem with that approach is that old binaries would not be able to take advantage of the technique; a re-linking would be required. 

Then the question arises, how does that bitmap get initialized? Drepper [suggests that systemtap be used](<https://lwn.net/Articles/275064/>): 

To fill in the bitmaps one can have separate a separate tool which is explicitly asked to update the bitmap data. To collect the page fault data one could use systemtap. It's easy enough to write a script which monitors the minor page faults for each binary and writes the data into a file. The binary update tool and can use the information from that file to generate the bitmap. 

Kleen's patch walks the page tables for a process when it is exiting, setting a bit in the bitmap if that page has been faulted in. Drepper sees this as suboptimal: 

Over many uses of a program all kinds of pages will be needed. Far more than in most cases. The prefetching should really only cover the commonly used code paths in the program. If you pull in everything, this will have advantages if you have that much page cache to spare. In that case just prefetching the entire file is even easier. No, such an improved method has to be more selective. 

The problem is in finding the balance between just prefetching the entire executable--which might be very wasteful--and prefetching the subset of pages that are most commonly used. It will take some heuristics to make that decision. As Drepper points out, recording the entire runtime of a program ""will result in all the pages of a program to be marked (unless you have a lot of dead code in the binary and it's all located together)."" 

The place where Drepper sees a need for kernel support is in providing a bitmap interface to `madvise()` so that any holes in the pages that get prefetched do not get filled by the readahead mechanism. The current interface would require a call to `madvise()` for each contiguous region, which could be add up to a large number of calls. Both he and Morton favor the bulk of the work being done in user space. 

Overall, there is lots more work to do before "predictive bitmaps" make their way into a Linux system--if they ever do. To start with, some benchmarking will have to be done to show that performance improves enough to consider making a change like this. David Miller expresses some [pessimism about the approach](<http://lwn.net/Articles/275088/>): 

I wrote such a patch ages ago as well. 

Frankly, based upon my experiences then and what I know now, I think it's a lose to do this. 

It is an interesting idea though, one that will likely crop up again if this particular incarnation does not go anywhere. Since the biggest efficiency gain is from reducing seeks, though, it may not be interesting long-term. As Morton [says](<http://lwn.net/Articles/275087/>), ""solid-state disks are going to put a lot of code out of a job.""
