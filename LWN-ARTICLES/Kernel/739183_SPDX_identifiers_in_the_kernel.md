---
title: SPDX identifiers in the kernel
url: https://lwn.net/Articles/739183/
date: "November 16, 2017"
category: Copyright issues
author: "By Jonathan Corbet November 16, 2017"
---

By **Jonathan Corbet**  
November 16, 2017

Observers of the kernel's commit stream or mailing lists will have seen a certain amount of traffic referring to the addition of SPDX license identifiers to kernel source files. For many, this may be their first encounter with SPDX. But the SPDX effort has been going on for some years; this article describes SPDX, along with why and how the kernel community intends to use it. 

On its face, compliance with licenses like the GPL seems like a straightforward task. But it quickly becomes complicated for a company that is shipping a wide range of software, in various versions, in a whole set of different products. Compliance problems often come about not because a given company wants to flout a license, but instead because that company has lost track of which licenses it needs to comply with and for which versions of which software. SPDX has its roots in [an effort that began in 2009](<https://lwn.net/Articles/370308/>) to help companies get a handle on what their compliance obligations actually are. 

It can be surprisingly hard to determine which licenses apply to a given repository full of software. The kernel's COPYING file states that it can be distributed under the terms of version 2 of the GNU General Public License. But many of the source files within the kernel tell a different story; some are BSD licensed, and many are dual-licensed. Some carry an exception to make it clear that user-space programs are not a derived product of the kernel. Occasionally, files with GPL-incompatible licenses have been found (and fixed). 

A great many files in the kernel source tree carry no license text at all. One might presume that these files are covered by GPLv2 but, as we'll see, the situation may not be quite that simple. No-license files are also problematic because the [Developer Certificate of Origin](<https://developercertificate.org/>), which governs contributions to the kernel, refers explicitly to ""the open source license indicated in the file"". If there _is_ no license indicated in the file, the meaning of that phrase is not entirely clear. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Another complicating factor is that the license text in kernel source files, when it is present at all, is entirely free-form. There are hundreds of variants of the GPLv2 text alone. That can make it hard for human readers to figure out what's going on, but it is even more challenging for software. It is not currently possible to run a tool on the kernel repository (or that of many other projects) and get a definitive list of the operative licenses. 

The [Software Package Data Exchange (SPDX)](<https://spdx.org/>) standard is an attempt to address this aspect of the licensing problem. This effort, which has come under the umbrella of the Linux Foundation's compliance program, has defined a way to declare licensing information that is intended to be easily read by both humans and machines. At its core, SPDX defines a single-line string to specify the license governing a file. It looks something like: 

```
SPDX-License-Identifier: GPL-2.0
```

There is [a long list of known licenses](<https://spdx.org/licenses/>) and the ability to add extra conditions or exceptions where needed. If each file in a repository contains one of these strings, summing up the licensing information for the repository as a whole becomes a straightforward affair. 

SPDX has been [adopted](<https://lwn.net/Articles/568286/>) in various parts of the industry in recent years. The effort to add SPDX identifiers to the kernel has been playing out, mostly in private, for at least a couple of years. It recently surfaced in the form of [a huge patch set](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ead751507de86d90fa250431e9990a8b881f713c>) adding SPDX identifiers to over 12,000 kernel source files that did not have any license information at all, and as [a brief discussion](<https://lwn.net/Articles/738235/>) at the 2017 Maintainers Summit. Somewhat later, [some documentation](<https://lwn.net/Articles/738809/>) on the project surfaced. Fully documenting the kernel with SPDX tags will take a while, but the process is well underway at this point. 

For kernel source files, the decision was made that the SPDX tag should appear as the first line in the file (or the second line for scripts where the first line must be the `#!` string). For normal C source files, the string will be a comment using the "`//`" syntax; header files, instead, use traditional (`/* */`) comments for reasons related to tooling. Thus, for example, if one looks at [`arch/alpha/include/uapi/asm/a.out.h`](<http://elixir.free-electrons.com/linux/v4.14/source/arch/alpha/include/uapi/asm/a.out.h>), one will see at the top: 

```
/* SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note */
```

The `WITH` string says that the kernel's user-space exception applies to this file, since it defines part of the system-call ABI. 

Kernel developers are often short of patience for things that look like bureaucratic exercises, so it would not have been surprising to see some opposition to this project. In truth, there has been little. The biggest issue would appear to be that some of the no-license files that were marked as GPLv2 [should maybe carry a different license](<https://lwn.net/Articles/739190/>). One could argue that this kind of disagreement is a good thing, in that it points out a place where the license applying to a specific file was not what most people might expect. Once this kind of problem comes to light, it can be addressed. 

The plan is to eventually have SPDX tags in all kernel source files, but that process could take some time. For each file that already carries a license text, somebody has to look and ensure that the SPDX tag matches that text exactly. Given that there are around 60,000 files in the kernel repository, that's a fair amount of work. An additional goal is to eventually get rid of the other license texts; the consensus seems to be that the SPDX identifier is a sufficient declaration of the license on its own. But removing license text from source files must be done with a great deal of care, so it may be a long time before anybody works up the courage to attempt that on any files that they do not themselves own the copyright for. 

It would not be surprising to see the process of adding SPDX tags extend over years. There will likely be an occasional flare-up as this work uncovers files with ambiguous or uncertain licensing, but that should result in more clarity around the licensing of the kernel as a whole once things are worked out. At the end, perhaps we'll know what the kernel's license story really is.
