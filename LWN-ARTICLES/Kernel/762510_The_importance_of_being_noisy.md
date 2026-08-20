---
title: The importance of being noisy
url: https://lwn.net/Articles/762510/
date: "August 13, 2018"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet August 13, 2018"
---

By **Jonathan Corbet**  
August 13, 2018

Hundreds (at least) of kernel bugs are fixed every month. Given the kernel's privileged position within the system, a relatively large portion of those bugs have security implications. Many bugs are relatively easily noticed once they are triggered; that leads to them being fixed. Some bugs, though, can be hard to detect, a result that can be worsened by the design of in-kernel APIs. A proposed change to how user-space accessors work will, hopefully, help to shine a light on one class of stealthy bugs. 

Many system calls involve addresses passed from user space into the kernel; the kernel is then expected to read from or write to those addresses. As long as the calling process can legitimately access the addressed memory, all is well. Should user space pass an address pointing to data it should not be able to access — a pointer into kernel space, for example — bad things can happen. 

The kernel protects itself against erroneous (or malicious) addresses from user space via a two-step mechanism. The first of these is the `access_ok()` macro: 

```
int access_ok(type, address, size);
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

This function will return a nonzero value if an access of the given `type` (`VERIFY_READ` or `VERIFY_WRITE`) to `size` bytes of memory at `address` makes sense — is that region of memory in a part of the address range that user space should be accessing? On most architectures, its job is to filter out attempts to access memory that is in kernel space. If `access_ok()` returns zero, no attempt to dereference the given address should be made. Otherwise, once that test is passed, the second step is to use one of any of a number of primitives to actually copy memory between user and kernel space, using normal memory protections to prevent unauthorized access. 

While most of the interfaces provided inside the kernel for access to user-space memory combine those steps, there are some that deliberately separate them, usually as a way to optimize several accesses happening in a row. When those interfaces are used, it is possible for the developer to forget to call `access_ok()` in one or more paths, leading to a situation where the kernel will access kernel-space memory using an address controlled by user space — never a good idea. That results in vulnerabilities like [CVE-2017-5123](<https://lwn.net/Articles/736348/>) or the recent [bsg problems](<https://lwn.net/Articles/760345/>). 

Many problems that cause the kernel to try to dereference a wild pointer can be flushed out by fuzzing. But, when the kernel's user-space access functions are asked to copy data to or from the wrong place, they simply return an `EFAULT` status that is silently passed back to the user-space caller. Most of the time, that is the right thing to do, since the most likely explanation is a bug in the user-space program. It may have asked to copy data from a portion of its address space that isn't mapped, for example, or to write to some read-only memory. 

The same thing happens, though, if user space asks the kernel to copy data to or from a random kernel-space address. Normally, the `access_ok()` call will catch the problem and no attempt to copy is made. But if `access_ok()` isn't called, the kernel may attempt to access kernel space on behalf of the user. In the absence of a focused attack, a random kernel-space address has a high probability of pointing to memory with no mapping at all, on a 64-bit system at least. The resulting page fault gets turned into an `EFAULT` return that is indistinguishable from any other error. 

If somebody is running a fuzzing program in user space, this `EFAULT` return will completely mask the fact that the kernel just tried to do something bad. So developers will remain unaware of the existence of the bug which, consequentially, will not be fixed. Eventually somebody else will discover it; that somebody may not have any interest in seeing the hole closed. 

This outcome is unfortunate because the kernel has all of the information it needs to know that a potentially severe security bug exists. With just a tiny number of exceptions, the user-space access functions will never be called with a pointer into kernel space. So if one of those functions generates a kernel-space page fault, something has gone wrong somewhere. It would make sense to try to draw attention to the problem so it can be investigated and fixed. 

That is the conclusion reached by Jann Horn, resulting in [this patch set](<https://lwn.net/ml/linux-kernel/20180807012257.20157-1-jannh@google.com/>) for the x86 architecture. The objective is simple: if a user-space access function faults with a kernel-space address, and the call site has not been specially marked, a `WARN()` call will result. That will create a kernel oops and a traceback in the kernel log. That should attract attention in many settings, but it is especially likely to when fuzzers are being run, since they are on the lookout for just that kind of result. 

The reaction to the patch set was uniformly positive; there were requests for various improvements, of course, but everybody seems to want to see this work proceed. 

Kernel developers tend to be careful not to send too much information to the system log. At best, excessive chattiness can make it hard to see the messages that are actually important; at worst, it can be exploited by the user to overflow the log or as a general denial-of-service attack. But the kernel, as it is now, is masking some important information about severe bugs that it should be able to detect. That silence should soon come to an end; sometimes making a little noise is exactly the right thing to do.
