---
title: "Kernel Summit: Security"
url: https://lwn.net/Articles/94572/
date: "July 21, 2004"
category: "Modules-Signed; Security"
author: 
---

This article is part of LWN's [2004 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2004/>).   
---  
James Morris led a session on security. He noted that a great many security features have found their way into 2.6; these include the Linux security module mechanism, the crypto API, the dm-crypt target, IPSec, SELinux, NX bit support, the audit framework, and more. Still, there are things yet to be done. These include: 

  * In-kernel keyring management. So far, the kernel has had little need to manage cryptographic keys in its own right, but that is likely to change. A simple patch has been posted, more work remains to do. 

  * An audit framework. The lightweight framework recently merged into 2.6 is a step in the right direction, but there is apparently more to do. It would be best if all distributions used the same framework; it would make certification easier. 

  * SELinux has some performance issues, especially in the networking area. The problems seem reasonably well understood, and ideas for solutions are being kicked around. 

  * SELinux also apparently needs multi-level security for groups dealing in classified data and similar materials. One might be forgiven for thinking that SELinux is already sufficiently complex, but it would seem that more complexity is required. 

  * "Labeled networking" is another wishlist item; it would allow packets to be marked on entry to a network and handled according to those labels. 

  * The integration of resource management code, presumably the class-based resource management mechanism. 

  * Virtualization work, allowing the complete isolation of processes running on virtual machines. 

  * Extension of the crypto API to support hardware encryption devices. 

  * Signed modules and binaries. The [signed module patch](<http://lwn.net/Articles/92617/>) is in circulation, and is part of the Fedora test release; signed binaries are further away. Linus asked if any developers were worried about the implications of this work, but nobody raised any complaints. 

  * Support for "trusted computing" platforms. 

  * "Better capabilities"; what "better" means was not really specified. It was noted that nobody is using the existing capability mechanism, which, until recently, did not even work very well. 

[>> Next: Class-based Kernel Resource Management](<https://lwn.net/Articles/94573/>).
