---
title: A module for crashing the kernel
url: https://lwn.net/Articles/371320/
date: "January 26, 2010"
category: Development tools
author: "By Jonathan Corbet January 26, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 26, 2010

Normally, a kernel which doesn't crash is considered to be a good thing. It can be a source of true frustration, though, for those who want to see the system go down in flames. The reliability of the system means that somebody waiting for a crash may grow old indeed in the process. 

Simon Kagstrom has heard the pain expressed by such users; in response, he has posted [a kernel module](<http://lwn.net/Articles/371208/>) just for people who want to be able to destroy their systems on demand. This module creates a directory (`provoke_crash`) in debugfs, filled with a number of useful files. For those with simple needs, a write to `bugon` results in a straightforward `BUG()` call. Users with more discriminating tastes can write to `null_dereference` to cause a null pointer dereference, `overwrite_allocation` to write beyond a heap allocation, or `corrupt_stack` to overwrite the stack. And truly kinky users can go for `oops_interrupt_context` to get a null dereference in softirq mode, `write_after_free` to step on freed memory, or `unaligned_load_store` to perform badly-aligned memory operations. 

Needless to say, this isn't a module one would ordinarily want to leave loaded into a production system; it's better kept in a secret place and pulled out after the kids go to sleep. Unless, of course, you have a real use for it; Simon has been employing it to make sure that [`kmsg_dump()`](<http://lwn.net/Articles/366987/>) does the right thing in various crash scenarios. For most developers, though, work is normally dominated by the need to _avoid_ crashes; since they'll have little use for this feature, it's not clear that this little module will ever make its way into the mainline.
