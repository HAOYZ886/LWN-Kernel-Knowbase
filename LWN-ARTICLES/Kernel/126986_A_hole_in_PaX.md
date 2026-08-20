---
title: A hole in PaX
url: https://lwn.net/Articles/126986/
date: "March 9, 2005"
category: "Security-Vulnerabilities"
author: 
---

Security software is, as a general rule, supposed to make a system more secure. So it is always discouraging when security code, instead, opens up new holes. The [PaX](<http://pax.grsecurity.net/>) patches are intended to harden the Linux kernel against various sorts of attacks; its developers have, at times, been quite harsh in their criticism of security in the mainline kernel. But, as [this advisory](<https://lwn.net/Articles/126987/>) shows, the PaX code, too, is not without its troubles. 

One of the techniques used by PaX is [VMA mirroring](<http://pax.grsecurity.net/docs/vmmirror.txt>). The PaX code tries to defeat various types of code injection attacks by completely separating the instruction and data areas of memory as seen by Linux processes. The idea is that, even if an attacker is able to overrun a buffer and direct the processor to execute the resulting code, the attack will be foiled by the processor's segmentation hardware. Any part of memory which can be accessed via a data pointer is simply not accessible as code. 

The problem is that some code segments in an executable file contain data as well - constant strings and such. So, when an executable ELF section is mapped into the code segment, it must also be "mirrored" in the data segment. This mirroring is accomplished by creating a special sort of virtual memory area (VMA) which refers to the same physical pages and backing store as code VMA, but which resides in the data portion of the address space. 

The details of the exploit have not yet been released. From a quick reading of the PaX patches before and after the fix, it would appear that the PaX code did not adequately restrict the changes user space could make to the mirrored VMAs. The resulting inconsistencies in the kernel's representation of the address space could then be exploited to run arbitrary code. 

The advisory notes that this vulnerability ""...pretty much destroys what PaX has always stood and been trusted for."" So the author is taking his marbles and going home; PaX will be discontinued at the end of this month. Certainly, introducing an exploitable hole into a security-related patch, where it lurked for a year and a half, could harm the trust users have in that patch. But giving up and leaving those users completely unsupported into the future seems likely to cause rather more damage. Bugs happen, even in the most carefully-written code. The best thing to do is to fix them and get on with life.
