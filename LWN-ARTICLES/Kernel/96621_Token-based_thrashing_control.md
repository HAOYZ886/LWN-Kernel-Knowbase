---
title: "Token-based thrashing control"
url: https://lwn.net/Articles/96621/
date: "August 4, 2004"
category: "Memory management-Token-based thrashing control; Thrashing"
author: 
---

A system which is in the throes of VM thrashing is no fun to work with. The kernel is forever throwing out pages which it will need in the near future in favor of pages needed right now, and little work actually gets done. It seems like there has to be a better way. 

Rik van Riel has put together [a patch](<https://lwn.net/Articles/95925/>) based on the work of Song Jiang which might help. The basic idea is that a process which is currently bringing in pages should, for a short period, not have its other pages booted out to swap. With luck, that process will actually make some progress during that grace period before the VM grim reaper swoops down and consigns it, once again, to the swap ghetto. 

Clearly, not all processes which are bringing in pages can be sheltered from page reclamation at the same time; if they could, the system would not be thrashing in the first place. This problem is addressed through the creation of a "swap token." A process holding the swap token will be allowed to bring in pages without having its current working set molested for a period of time. After a while, the token is passed on to the next needy process. 

In Rik's patch, the (single, system-wide) token is implemented through `swap_token_mm`, a pointer to the `mm` structure of the process holding the token. If the kernel, on behalf of a process incurring a page fault, decides that the token is available, `swap_token_mm` will be set and the faulting process will get its breathing space for a while. The token is deemed to be available if (1) it has been held for longer than the maximum period, which is set to a surprisingly long 300 seconds, or (2) the process holding the token has not incurred any page faults recently. Once the token becomes available, the first process which comes looking for it will grab it - unless it has held the token in the recent past. 

Rik's tests show some performance improvements with this patch applied. There are potential improvements which could be made, such as trying to add some intelligence to the decision of which process gets the token. A huge process may hold the token for some time, grow to fill much of memory, and still not have enough to get any real work done. Meanwhile, small processes which could have benefited from a few extra pages continue to thrash. Some tweaks could be made to the patch to address this issue, but there are limits to how much code and complexity should be added to the kernel to deal with a (hopefully) rare situation.
