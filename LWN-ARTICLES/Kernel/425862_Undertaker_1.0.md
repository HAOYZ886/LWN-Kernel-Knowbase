---
title: Undertaker 1.0
url: https://lwn.net/Articles/425862/
date: "February 1, 2011"
category: "Development tools-Undertaker"
author: "By Jonathan Corbet February 1, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
February 1, 2011

As anybody who has ever sat through an "allyesconfig" build - or a build using a distributor configuration - understands, there is a _lot_ of code in the kernel. Most of the time, creating the perfect kernel is a matter of excluding code until the size becomes reasonable. So it's a rare kernel build that actually compiles a majority of the code found in the tree. 

The kernel configuration mechanism makes it possible to perform this selection. Part of this mechanism is wired into the build system; it allows source files to be passed over entirely if they contain nothing of interest. The other half, though, is implemented with preprocessor symbols and conditional compilation. Kernel developers may be discouraged from using `#ifdef`, but there are still a lot of conditional blocks in the code. 

Sometimes, the logic which leads to the inclusion or exclusion of a specific block is complex and not at all clear. There are many configuration options in the kernel, and they can depend on each other in complicated ways. As a result, dead code - code which will not be compiled regardless of the selected configuration - may escape notice for years. Dead code adds noise to the source tree and, since nobody ever runs it, it is more than likely to contain bugs. If that code is re-enabled or copied, those bugs could spread through the tree in surprising ways. 

So it would be good to be able to identify dead code and get it out of the tree. The newly-released [undertaker](<https://lwn.net/Articles/425241/>) tool was designed to do a number of types of static analysis, including dead code identification. Developers can run it on their own to find dead blocks in specific files; there is also [a web interface](<http://www4.informatik.uni-erlangen.de/Research/VAMOS/linux-trees/>) which allows anybody to browse through the tree and find the dead sections. That should lead to patches hauling away the bodies and cleaning up the tree, which is a good thing.
