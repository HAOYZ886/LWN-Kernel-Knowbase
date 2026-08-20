---
title: Memory allocation failures and scary warnings
url: https://lwn.net/Articles/276731/
date: "April 7, 2008"
category: "Development model-Kernel quality"
author: "By Jonathan Corbet April 7, 2008"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 7, 2008

People who put their Linux systems under a certain amount of memory stress \- and who look at their logfiles - may notice an occasional message indicating that a "page allocation failure" has occurred, followed by a scary backtrace. These people may also notice that, despite the apocalyptic appearance of this message, the world often fails to end. In fact, the system tends to carry on just fine. For this reason, Dave Jones, who probably gets ten emails for every backtrace generated on a Fedora system, has [suggested](<https://lwn.net/Articles/276733/>) that these messages are simply noise which should be removed. Whether that should really happen is not entirely clear, though; understanding why requires a bit of background. 

In general, the kernel's memory allocator does not like to fail. So, when kernel code requests memory, the memory management code will work hard to satisfy the request. If this work involves pushing other pages out to swap or removing data from the page cache, so be it. A big exception happens, though, when an atomic allocation (using the `GFP_ATOMIC` flag) is requested. Code requesting atomic allocations is generally not in a position where it can wait around for a lot of memory housecleaning work; in particular, such code cannot sleep. So if the memory manager is unable to satisfy an atomic allocation with the memory it has in hand, it has no choice except to fail the request. 

Such failures are quite rare, especially when single pages are requested. The kernel works to keep some spare pages around at all times, so the memory stress must be severe before a single-page allocation will fail. Multi-page allocations are harder, though; the kernel's memory management code tends to fragment pages, making groups of physically-contiguous pages hard to find. In particular, if the system is under pressure to the point that there is not much free memory available at all, the chances of successfully allocating two (or more) contiguous pages drops considerably. 

Multi-page allocations are not often used in the kernel; they are avoided whenever possible. There are situations where they are necessary, though. One example is network drivers which (1) support the transmission and reception of packets too large to fit into a single page, and which (2) drive hardware which cannot perform scatter/gather I/O on a single packet. In this situation, the DMA buffers used for packets must be larger than one page, and they must be physically contiguous. This is a situation which will become less pressing over time; scatter/gather capability in the hardware is increasingly common, and drivers are being rewritten to make use of this capability. With sufficiently smart hardware, the need for multi-page allocations goes down considerably. 

But all of that skirts around the main point, which is that kernel code is supposed to handle allocation failures properly. There is never any guarantee that memory will be available, so kernel code must be written defensively. Allocation failures must be handled without losing any more capability than is strictly necessary. If one assumes that kernel code is written correctly, there should be no need to issue warnings on allocation failures. Things should just continue to work, perhaps without users noticing at all. 

And, in fact, things often do just work. But the discussion resulting from Dave's suggestion makes it clear that few developers are confident that all kernel code does the right thing in the face of memory allocation problems. In cases where an allocation failure is _not_ dealt with correctly, the system may go down in random places, leaving few clues as to what really happened. In that kind of situation, the allocation failure warning may be the only useful information which survives the crash. For this reason, some people want to see the warnings left in place. 

As it happens, the memory allocator supports a special bit (`__GFP_NOWARN`) which causes the warning not to be emitted if a specific allocation fails. So it has been suggested that the allocations made from code which is known to handle failures properly have `__GFP_NOWARN` set. That would kill the warnings in code known to do the right thing while leaving it for all other callers, presumably limiting the warnings to places where there might truly be a problem. Jeff Garzik [strongly opposed this idea](<https://lwn.net/Articles/276734/>), though, saying that it clutters up the code and "punishes good behavior." 

The other reason given for keeping the warnings in place is to make it clear when a system is running under persistent memory pressure. Such systems will not be performing optimally; often there are changes which can be made to relieve the pressure and help the system to run more smoothly. So it has been suggested that the warning could be reduced in frequency and made less scary. Nick Piggin [suggests](<https://lwn.net/Articles/276735/>): 

So I think that the messages should stay, and they should print out some header to say that it is only a warning and if not happening too often then it is not a problem, and if it is continually happening then please try X or Y or post a message to lkml... 

An alternative idea would be to keep some sort of counter somewhere which could be queried by curious system administrators. 

Of course, the _real_ solution is to ensure that all kernel code is robust in the face of allocation failures. This can be hard to do, since the error recovery paths in any code are not often exercised or tested. Fortunately, the [fault injection framework](<http://lwn.net/Articles/209257/>) can help in this situation. Kernel developers can use this framework to simulate allocation failures in specific regions of code, then watch to see what happens. Your editor's impression, though, is that relatively few developers are using this tool. So confidence in the kernel's handling of allocation failures may remain low, and the desire to keep the warning around may remain high.
