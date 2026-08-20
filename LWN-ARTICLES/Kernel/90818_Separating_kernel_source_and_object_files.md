---
title: Separating kernel source and object files
url: https://lwn.net/Articles/90818/
date: "June 23, 2004"
category: Build system
author: 
---

The build process in recent 2.6 kernels allows for the separation of source and object trees. If a kernel build is started with the `O=` option, the resulting object files (and other built files) will go into the directory specified, rather than being mixed in with the source. Some developers find this way of doing things easier to manage, especially if the same source tree is being used to build kernels for multiple architectures or with multiple sets of configuration options. 

One distributor (SUSE) has begun shipping kernels which have been built in this manner. The difference has gone unnoticed by almost all users, but one vendor of proprietary modules recently posted [a strong message](<https://lwn.net/Articles/90821/>) accusing SUSE of forking the kernel. The specific issue is that this vendor's modules would no longer build with SUSE's kernels, and that problem turned out to be a result of the separated source and object trees. 

When a kernel's modules are installed under `/lib`, a symbolic link called `build` is made pointing to the source tree. This link is used by the external module build process to find kernel headers, configuration files, and needed object files. When SUSE adopted the separate object directory, it redirected the `build` link to point to that directory, rather than to the original source. That is, after all, where many of the necessary files will be found. Unfortunately for this particular vendor, their modules needed some other files which are only found in the source tree. When the `build` link was directed elsewhere, those modules would no longer compile. 

The fix was relatively straightforward, but this situation forced a new discussion on how the build system should work when separate object directories are in use. The result is [a new patch](<https://lwn.net/Articles/90689/>) from Sam Ravnborg which nails down how these links should work. With this patch (not merged as of this writing), the `build` link would always point to the object directory. Doing things this way allows most external modules to continue to build without changes. A new link (`source`) will be added to point to the source directory when needed. And a small, special-purpose makefile is placed in the object directory; its job is to bridge the gap between the two trees and make most external module builds work with no changes required.
