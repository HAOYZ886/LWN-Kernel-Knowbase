---
title: x86 NX support
url: https://lwn.net/Articles/87814/
date: "June 2, 2004"
category: "NX support; Security-Security technologies"
author: 
---

Marking regions of memory as not containing executable code is not a particularly new technique; some processors have recognized this mode for years. The processor that everybody actually _uses_ , however (the x86 family) does not have a "no-execute" bit. 

At least, it didn't until very recently. AMD added a no-execute (NX) permission bit to the page table entries in its 64-bit processors; Intel has recently said it will be supporting this mode as well. So the hardware will be able to avoid executing code from certain regions of memory, making various types of buffer overflow attacks harder. At least, that will be true if the operating system supports and uses the NX mode. 

To that end, Ingo Molnar has [posted a patch](<https://lwn.net/Articles/87808/>) bringing NX support to the x86 architecture; his patch is based on previous work done by Intel and the x86_64 NX support by Andi Kleen. This patch allows applications to mark areas as being non-executable; such areas, typically, will include the stack and heap zones. It also applies the NX bit to the kernel itself; kernel text is marked executable, but kernel data is not. As a result, the next time a buffer overflow turns up in the kernel, it, too, will be harder to exploit. 

The NX bit only works when the processor is running in the PAE mode. Most x86 Linux systems currently do not run in that mode; it is normally only turned on when large amounts of memory (more than 4GB) are installed. This mode adds a third level of page tables, and makes the page table entries themselves larger, so users and distributors normally turn it off if it is not needed. Most modern x86 processors support the PAE mode, however; security considerations may lead to it being used more heavily in the future. 

Linus's main [concern](<https://lwn.net/Articles/87815/>) about the patch would appear to be how many old applications it might break. The [reply](<https://lwn.net/Articles/87816/>) from Arjan van de Ven is that pretty much everything "just works." The no-execute permission is not applied unless the code is specially marked in the image file, and gcc apparently does a good job of not setting that flag when it would break things. If this experience holds true, NX support could go in fairly quickly, and a longstanding x86 security weakness will be no more. 

For people interested in testing this patch, Arjan has merged it into the latest Fedora Core test kernels. See [the patch announcement](<https://lwn.net/Articles/87808/>) for a pointer. There is also [a "quickstart" document](<http://people.redhat.com/mingo/nx-patches/QuickStart-NX.txt>) for those who would like to test out NX in their own kernels.
