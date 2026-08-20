---
title: Ksplice and kreplace
url: https://lwn.net/Articles/308409/
date: "November 24, 2008"
category: Ksplice; Live patching
author: "By Jonathan Corbet November 24, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 24, 2008

Rebooting a system to apply a security update is a pain. In some situations, it's more than a pain; for various reasons, many systems cannot be taken down at all without compromising the work they are supposed to be doing. Back in April, LWN [looked at Ksplice](<http://lwn.net/Articles/280058/>), a mechanism designed to enable the installation of kernel updates without the need to reboot the system. Since then, work has continued on Ksplice, [a new version](<http://lwn.net/Articles/308241/>) has been posted, and the project is starting to push toward mainline inclusion. So another look is called for. 

The core idea behind Ksplice remains the same: when given a source tree and a patch, it builds the kernel both with and without the patch and looks at the differences. To that end, the compilation procedure is modified to put every function and data structure into its own executable section. That makes life a little harder for the compiler and the linker, but developers are notably insensitive to the difficulties faced by those tools. With things split up this way, it is relatively easy to identify a minimal set of changes in the binary kernel image which result from the patch. Ksplice can then, with some care, patch the new code into the running kernel. Once this work is done, the old kernel is running the new code without ever having been rebooted. 

This technique works well for code changes, but different challenges come with changes to data structures. Back in April, Ksplice could not handle that kind of change. Even so, the project's developers claimed to be able to apply the bulk of the kernel's security updates using ksplice. Since then, though, the developers have applied some energy to this problem. With the addition of a couple of new techniques - which require extra effort on the part of the person preparing the patch for Ksplice - it is now possible to apply 100% of the 65 non-DOS security patches released for the kernel since 2005. 

In some cases, a kernel patch will simply require that a data structure be initialized differently. The way to handle this change in an update through Ksplice is to modify the relevant data structures on the fly. To effect such changes, a patch can be modified to include code like the following: 

```
#include <ksplice-patch.h>
    
        ksplice_apply(void (*func)());
```

While Ksplice is applying the changes - and while the rest of the system is still stopped - the given `func` will be called. It can then go rooting through the kernel's data structures, changing things as needed. For example, [CVE-2008-0007](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-0007>) came about as a result of a failure by some drivers to set the `VM_DONTEXPAND` flag on certain `vm_area_struct` structures. Ksplice is able to apply the fix to the drivers without trouble, but that is not helpful for any incorrectly-initialized VMAs present on the running system. So [the modifications to the patch](<http://www.ksplice.com/cve/cve-2008-0007-ksplice-changes.patch>) add some functions which set `VM_DONTEXPAND` on existing VMAs, then use `ksplice_apply()` to cause those functions to be executed. The result is a fully-fixed system. 

Changes to data structure definitions are harder. If a structure field is removed, the Ksplice version of the patch can just leave it in place. But the addition of a new field requires more complicated measures. Simply replacing the allocated structures on the fly seems impractical; finding and fixing all pointers to those structures would be difficult at best. So something else is needed. 

For Ksplice, that something else is a "shadow" mechanism which allocates a separate structure to hold the new fields. Using shadow structures is a fair amount of additional work; the original patch must be changed in a number of places. Code which allocates the affected structure must be modified to allocate the shadow as well, and code which frees the structure must be changed in similar ways. Any reference to the new field(s) must, instead, look up the shadow structure and use that version of the field. All told, it looks like a tiresome procedure which has a significant chance of introducing new bugs. There is also the potential for performance issues caused by the linear linked list search performed to find the shadow structures. The good news is that it is only rarely necessary to modify a patch in this way. 

The Ksplice developers do not appear to be done yet; from the latest patch posting: 

We're currently working on the problem of making it feasible to apply the entire stable tree using Ksplice. Although Ksplice's original evaluation focused on patches for CVEs, we understand the idea that "security bugs are just 'normal bugs'" (i.e., tracking security bugs separately from normal bugs can be difficult and isn't necessarily advisable). We ultimately want to provide to long-running machines hot updates for all of the bug fixes that go into the corresponding stable tree. 

This is an ambitious goal; a single stable series can add up to hundreds of changes, some of which can be reasonably large. It will be interesting to see how many users are really interested in this particular sort of update; sites running critical systems tend to have older "enterprise" kernels which are no longer receiving stable tree updates. But a Ksplice which is flexible enough to handle that kind of update stream should also be useful for distributors wanting to provide no-reboot patches to their customers. 

Meanwhile, Nikanth Karthikesan has posted a facility called [kreplace](<http://lwn.net/Articles/308236/>). On the surface, it looks similar to Ksplice, but the goal is a little different: its purpose is to allow a developer to quickly try out a change on a running kernel. Kreplace works by simply patching out and replacing one or more functions in the kernel. Kreplace may have its value, but the initial reaction has not been greatly enthusiastic. Among other things, it has been [pointed out](<https://lwn.net/Articles/308421/>) that Ksplice also has a facility to allow for quick experimentation with changes - though it will be quick only if the developer is already set up to use Ksplice with the running kernel. 

A final concern with either of these solutions is that they are, for all practical purposes, employing rootkit techniques. A mechanism which can be used by distributors to patch running systems can also be (mis)used by others. Vendors of binary-only modules could, for example, use Ksplice or kreplace to get around GPL-only exports and other inconvenient features of contemporary kernels. Crackers could also use it, of course, but they already have their own rootkit tools and gain no real benefit from an officially-supported runtime patching mechanism. Whether this aspect of Ksplice is of concern to the development community may be seen in the coming months as this code gets closer to mainline inclusion.
