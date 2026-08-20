---
title: Ext4 data corruption hits the stable kernels
url: https://lwn.net/Articles/954770/
date: "December 14, 2023"
category: "Development model-Stable tree; Filesystems-ext4"
author: "By Jonathan Corbet December 14, 2023"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 14, 2023

The kernel's stable-update process is intended to produce kernels that are, well, stable; when that promise is lived up to, users can update to newer stable updates without fear. By any account, a bug that corrupts data on ext4 filesystems constitutes a failure to hold to that promise. As is so often the case, this problem is the result of a chain of failures in a system that works well most of the time. 

For many people, the first notice of this problem came in the form of [this Debian bug report](<https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1057843>) stating that the 6.1.64 kernel, which had been rolled into the planned Debian 12.3 release, was exhibiting data-corruption problems. Debian developers quickly [put that release on hold](<https://lwn.net/Articles/954284/>) while investigating the problem, which was good as far as it went, but many Debian users update their systems often rather than waiting for the point releases. They may have gotten buggy kernels as a result. The bug is likely to have hit users of other fast-moving distributions and those who install their own kernels as well. 

As it happens, this bug had been found during the two-day comment period for the 6.1.64 kernel release (among others); Daniel Diaz [reported](<https://lwn.net/ml/linux-kernel/81a11ebe-ea47-4e21-b5eb-536b1a723168@linaro.org/>) the problem, which affected the planned 5.10.202 and 5.15.140 releases as well, in the 5.15.140 discussion thread on November 24\. Diaz had bisected the problem and reported the problematic commit, which was quickly removed from the 5.10.202 and 5.15.140 patch sets prior to the release of those kernels; for unknown reasons, almost certainly a simple mistake, the problematic patch remained in the 6.1.64 set, and was released with that kernel. 

The patch in question was written by Jan Kara, who did not manage to track down the specific problem until December 5, by which time 6.1.65, also buggy, had also been released. One might say that this shows a lack of urgency but, in Kara's defense, the patch clearly worked in the mainline and had been (most people thought) removed from the stable updates. Once he figured it out, he quickly [raised the alarm](<https://lwn.net/ml/linux-kernel/20231205122122.dfhhoaswsfscuhc3@quack3/>): 

> So all stable kernels that have 91562895f803 ("ext4: properly sync file size update after O_SYNC direct IO") before 6.5 are corrupting data - I've noticed at least 6.1 is still carrying the problematic commit. Greg, please take out the commit from all stable kernels before 6.5 as soon as possible, we'll figure out proper backport once user data are not being corrupted anymore. 

The [patch in question](<https://git.kernel.org/linus/91562895f803>) is a fix for a different ext4 bug, where a direct I/O operation could, after a badly timed crash, result in an incorrect file size. Kara's fix for the problem was applied to 6.7-rc1, and is by all appearances correct; there have been no reports of corruption problems on current mainline kernels. 

Unfortunately, that patch depended on [a different patch](<https://git.kernel.org/linus/936e114a245b6>) from Christoph Hellwig that first appeared in 6.5-rc1. That patch made a subtle change in how the file position in an open file was updated after a direct I/O operation, causing that update to happen a bit later than before. That patch, too, is correct. But Kara's patch, in the absence of Hellwig's earlier patch, was not. The file position of a file under direct I/O was not updated properly, causing subsequent writes to end up in the wrong place and corrupting the file. 

It is worth noting that, as [Kara later clarified](<https://lwn.net/Articles/954841/>), a specific sequence of operations is required to hit this particular bug. It may well be that no real-world application, other than the test suite that revealed it, was affected. 

Once the problem was identified, Greg Kroah-Hartman [responded](<https://lwn.net/ml/linux-kernel/2023120643-evade-legal-ee74@gregkh/>) that a fixed update would be produced. Interestingly, Kroah-Hartman did not remove the problematic patch as Kara had requested; instead, he backported Hellwig's prerequisite patch to 6.1.x as well. Arguably this was a riskier approach, but it did have the advantage of preserving the ext4 fix that was the objective of Kara's patch. The newly backported patch appeared in the [6.1.66 stable update](<https://lwn.net/ml/linux-kernel/2023120858-implant-detonate-edbc@gregkh/>) that appeared, with no special fanfare, on December 8 — 15 days after the buggy 6.1.64 update. 

There are clearly (at least) two things that went wrong here. One of those was the decision to backport Kara's patch without addressing its dependency on Hellwig's 6.5 change. That kind of dependency can be difficult to notice in kernel code, though. There were no API changes that would have allowed a compiler to raise an error. Perhaps if a language like Rust were in use, and if the iomap abstractions were written tightly enough that this sort of semantic change was reflected in the API, the problem would not have escaped notice. But even Rust is unable to guarantee a lack of logic errors. 

The other mistake, of course, was releasing 6.1.64 with the bug in it. This error has led to a new round of complaints from some about the kernel community's lack of testing infrastructure. While there is no doubt that kernel testing could stand significant improvement, it must be pointed out that this bug was, in fact, caught and identified by testing, and reported on the list, before the release was made. The problem was that, while the buggy backport was removed from 5.10.202 and 5.15.140, it was not taken out of 6.1.64. 

One can certainly try to figure out how that went wrong in the hopes of preventing a similar episode in the future. It's hard to look past the fact that Kroah-Hartman received a report that 6.1.64 had a problem, but failed to act on it. That said, the report was made in response to the 5.15.140-rc1 posting; had there been a separate report made in response to 6.1.64-rc1 as well, it might have been more difficult to overlook. The kernel community's lack of a proper issue-tracking system certainly will not have helped here. In the end, though, the truth of the matter is that, even in 2023, stable kernel updates are made by humans, and humans occasionally make mistakes. 

Even one file-corruption bug is too many, of course, and this is not the first bug of this type that has hit the stable updates. That has predictably led to calls for the stable update process to be far more selective in the patches that are backported. The problem with that idea, of course, is that it is expensive to implement (and volunteers to do the work are scarce), and it tends to be reactive, only backporting fixes after somebody has been bitten by a bug. 

The small team of developers working on the stable updates has decided, instead, to backport as many fixes as possible. The claim, seemingly backed up by studies done by the Android team, is that this approach gets fixes to users before the problems they fix (such as security vulnerabilities) make themselves known. Arguably, the end result is kernels with fewer problems overall, even though the backporting process may occasionally introduce one, as happened here. But it does lead to more churn in the stable updates than many users would like. 

Ways to improve the situation are clear; they mostly involve more testing of stable releases and more eyes on the patches that land there. That requires the contribution of resources — machines, software, and people — to the effort. In the end, the best way to get better stable kernel updates is to help make them better.
