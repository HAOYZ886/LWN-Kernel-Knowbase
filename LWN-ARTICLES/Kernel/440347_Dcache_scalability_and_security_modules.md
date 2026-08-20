---
title: Dcache scalability and security modules
url: https://lwn.net/Articles/440347/
date: "April 27, 2011"
category: "Dentry cache; Scalability; Security-Security modules"
author: "By Jonathan Corbet April 27, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 27, 2011

The [dentry cache scalability patch set](<https://lwn.net/Articles/419811/>) was merged for the 2.6.38 kernel; it works by attempting to perform pathname lookup with no locks held at all. The read-copy-update (RCU) mechanism is used to ensure that dentry structures remain in existence for long enough to perform the lookup. This patch set has removed a significant scalability problem from the kernel, improving lookup times considerably. Except, as it turns out, it doesn't always work that way. A set of patches merged for 2.6.39-rc5 - rather later in the development cycle than one would ordinarily expect - has helped to address this problem. 

The fact that the pathname lookup fast path runs under RCU means that no operation can block. Should it turn out that the lookup cannot be performed without blocking (if a directory entry must be read from disk, for example), the fastpath lookup is aborted and the whole process starts over in the slow mode. In the 2.6.38 lookup code, the mere fact that security modules have been built into the kernel will force a fallback to slow mode, even if no actual security module is active. Things were done this way because nobody had taken the time to verify whether the security module `inode_permission()` checks were RCU-safe or not. So, if security modules are enabled, the result is not just that the scalability advantages over 2.6.37 are not available; in fact, the code runs _slower_ than it did in 2.6.37. 

Enterprise distributions have a tendency to enable security modules, so this performance problem is a real concern. In response, Andi Kleen took a look at the code and found that improving the situation was not that hard; his [patches](<https://lwn.net/Articles/439858/>) led to what was merged for 2.6.39. Andi started by allowing individual security modules to decide whether they could perform the inode permissions check safely in the RCU mode or not, with the default being to fall back to slow mode. Since the default `inode_permission()` check does nothing, it could easily be made RCU safe; with just that change, systems with security modules enabled but with no module active can make use of the fast lookup path. 

Looking further, Andi discovered that both SELinux and SMACK already use RCU for their permissions checking. Given that the code is already RCU-safe, extending it to do RCU-safe permission checks was relatively straightforward. The only remaining glitch is situations where auditing is enabled; auditing is not RCU-safe, so things will still slow down on such systems. Otherwise, though, the advantages of the dcache scalability work should now have been extended to systems with security modules enabled - assuming that the late-cycle patches do not result in regressions that cause them to be reverted.
