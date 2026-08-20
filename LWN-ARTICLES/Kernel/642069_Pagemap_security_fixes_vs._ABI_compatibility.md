---
title: "Pagemap: security fixes vs. ABI compatibility"
url: https://lwn.net/Articles/642069/
date: "April 29, 2015"
category: "Development model-User-space ABI; Memory management"
author: "By Jonathan Corbet April 29, 2015"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 29, 2015

The kernel development community maintains a strong commitment to ABI compatibility; as a general rule, changes that will break existing applications are not allowed. But the community is also committed to fixing known security problems. There are times when a security issue cannot be fixed without changing the way a user-visible interface works, and that can lead to problems. One such situation has come up as the result of a change merged for the 4.0 kernel. 

Back in 2008, the 2.6.25 kernel included [a patch](<http://git.kernel.org/linus/85863e475e59afb027b0113290e3796ee6020b7d>) adding a new virtual file (called `pagemap`) to each process's `/proc` directory. That file contains an array of 64-bit values describing each page in the process's virtual address space. If the page is currently resident, the physical page-frame number will be given; otherwise, information on how to find the page in swap is provided. The original purpose for the `pagemap` file was to enable investigations into which pages were resident and which were shared with other processes. [Documentation/vm/pagemap.txt](<https://lwn.net/Articles/642072/>) has information on what can be found in this file. 

At the time this patch was merged, there appeared to be no harm in exposing the physical page-frame information. Since then, though, sentiments have turned against disclosing internal kernel information that is not strictly needed by user space. That, alone, might have eventually inspired somebody to remove the page-frame number from the `pagemap` file but, as it happens, something else came along first. 

That something is the "[rowhammer vulnerability](<http://googleprojectzero.blogspot.com/2015/03/exploiting-dram-rowhammer-bug-to-gain.html>)," wherein the contents of a memory area can be changed by repeatedly hammering on a nearby memory area. If an attacker wanted to use this technique to compromise a system, the first order of business would be to obtain access to a page of memory physically adjacent to the memory that is targeted to be changed. The contents of the `pagemap` file, by providing the physical location of every page mapped in the system, would obviously be most helpful in such a situation. There will probably be other ways for an attacker to determine how pages are laid out in physical memory, but `pagemap` is almost certainly the easiest way. 

To make life harder for attackers attempting to exploit the rowhammer vulnerability, [a simple patch](<https://lwn.net/Articles/642074/>) was merged for the 4.0-rc5 release in March. The patch turned the `pagemap` file into a privileged interface; attempts to open it will now fail unless the process in question has the `CAP_SYS_ADMIN` capability. The 4.0 release came out with that restriction in place, and everybody who was paying attention slept a little easier. 

But that rest appears to have come at the cost of some sleepless nights elsewhere. It turns out that [the UndoDB debugger uses the `pagemap` file](<https://lwn.net/Articles/642075/>) to track changes to memory. When changes need to be tracked, the debugger will `fork()` the process, putting all of its writable memory into copy-on-write mode. After running the operation of interest (a system call, normally), the debugger can scan the `pagemap` file to see which pages have changed page-frame numbers; those are the pages that were written to, and, thus, copied. Without access to `pagemap`, UndoDB cannot get this information and, as a result, it no longer works. 

In some situations of this type, one might just argue that the tool in question should be run as root. But that is not generally a desirable way to run an interactive debugging tool. So some other sort of solution must be found, or UndoDB will remain broken. There are cases where "remains broken" may be the final outcome; as Linus [said](<https://lwn.net/Articles/642077/>) in response to the report, ""the one exception to the regression rule is 'security fixes'"". But, fortunately, there appear to be some better options available this time around. 

One possibility would be to restore access to the `pagemap` file but to somehow scramble the page-frame numbers before reporting them to user space. That would work for UndoDB, since it doesn't care about the actual page-frame numbers; it is only looking for changes. Linus was not convinced that this was the right way to go, though: 

However, I don't believe that we have a good enough scrambling model to make that reasonable. Remember: any attacker will be able to see our scrambling code, so it would need to be both cryptographically secure *and* use a truly random per-VM secret key. Quite frankly, that's a _lot_ of effort for dubious gain... 

Andy Lutomirski also [pointed out](<https://lwn.net/Articles/642078/>) that even scrambled page-frame numbers might be enough for an attacker to obtain some memory-adjacency information. So that approach does not appear to be viable. 

The alternative is to simply report the page-frame numbers as zero in the absence of `CAP_SYS_ADMIN`. That would make the rest of the information in `pagemap` available while not exposing the page-frame information. The bad news is that always-zero page-frame numbers are not helpful for UndoDB. The good news, though, is that there is something else in `pagemap` that is just as useful. 

That "something else" is the "soft-dirty" mechanism added to the 3.11 kernel in support of the [checkpoint-restore in user space (CRIU)](<http://criu.org/Main_Page>) effort. Along with the page-frame number, each `pagemap` entry contains a soft-dirty bit that is meant to track pages that have been written to. All of the soft-dirty bits for a process can be reset to zero by writing to the `clear_refs` file in that process's `/proc` directory. Thereafter, the soft-dirty bit will be set whenever that process writes to a given page. CRIU uses this mechanism to find pages that have been changed during the checkpoint process, but it also will work for the UndoDB case. (See [Documentation/vm/soft-dirty.txt](<https://lwn.net/Articles/642080/>) for details on the soft-dirty mechanism). 

So the probable outcome in this case is that `pagemap` will, once again, become globally readable. But it will contain no useful page-frame numbers unless the reading process had `CAP_SYS_ADMIN` when it opened the file. That will make UndoDB users happy again while preserving the security objectives of the original patch. So this story has a happy ending — unless, of course, another user who truly needs the page-frame number information steps forward.
