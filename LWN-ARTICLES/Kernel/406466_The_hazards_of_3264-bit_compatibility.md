---
title: "The hazards of 32/64-bit compatibility"
url: https://lwn.net/Articles/406466/
date: "September 22, 2010"
category: "Security-Vulnerabilities; User-space API"
author: "By Jake Edge September 22, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
September 22, 2010

A kernel bug that was found--and fixed--in 2007 has recently reared its head again. Unfortunately, the bug was reintroduced in 2008, leaving a rather large pile of kernel versions that are vulnerable to a local privilege escalation on x86_64 systems. Though perhaps difficult to do, it would seem that some kind of regression testing suite for the kernel might be able to detect these kinds of problems before they get released to the world. 

There are two semi-related bugs that are both currently floating around, which is causing a bit of confusion. One was originally [CVE-2007-4573](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2007-4573>), and was reintroduced in a cleanup patch in June 2008. The reintroduced vulnerability has been tagged as [CVE-2010-3301](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3301>) (though the CVE entry is simply reserved at the time of this writing). Ben Hawkes found a somewhat similar [vulnerability](<http://sota.gen.nz/compat1/>)--also exploiting system calls from 32-bit binaries on 64-bit x86 systems--which led him to the discovery of the reintroduction of CVE-2007-4573. 

There are numerous [pitfalls](<http://lwn.net/Articles/311630/>) when trying to handle 32-bit binaries making system calls on 64-bit systems. Linux has a set of functions to handle the differences in arguments and calling conventions between 32 and 64-bit system calls, but it has always been tricky to get right. What we are seeing today are two instances where it wasn't done correctly, and the consequences of that can be dire. 

The 2007 problem stemmed from a mismatch between the use of the `%eax` 32-bit register to store the system call number (which is used as an index into the syscall table) and the use of the `%rax` 64-bit register (which contains `%eax` as its low 32 bits) to do the indexing. In the "normal" system call path, `%eax` was zero-extended before the 32-bit system call number from user space was stored, but there was a second path into that code where the upper 32 bits in `%rax` were not cleared. 

The `ptrace()` system call has the facility to make other system calls (using the `PTRACE_SYSCALL` request type) and also gives a user the ability to set register values. An attacker could set the upper 32 bits of `%rax` to a value of their choosing, make a system call with a seemingly valid index (in `%eax`) and end up indexing somewhere outside of the syscall table. By arranging to have exploit code at the designated location, the attacker can get the kernel to run his code. 

The `ptrace()` path was [fixed by Andi Kleen](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=176df2457ef6207156ca1a40991c54ca01fef567>) in September 2007 by ensuring that `%eax` (and other registers) were zero-extended. But zero-extending `%eax` was removed in Roland McGrath's [clean up patch](<http://git.kernel.org/?p=linux/kernel/git/stable/linux-2.6.35.y.git;a=commitdiff;h=d4d67150165df8bf1cc05e532f6efca96f907cab>) in June 2008. When Hawkes and Robert Swiecki recently noticed the problem, they had little difficulty in [modifying an exploit](<http://sota.gen.nz/compat2/>) from 2007 to get a root shell on recent kernels. 

CVE-2010-3301 was resolved by a pair of patches. McGrath [put](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=eefdca043e8391dcd719711716492063030b55ac>) the zero-extension of the `%eax` register back into the ptrace path, while H. Peter Anvin [made](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=36d001c70d8a0144ac1d038f6876c484849a74de>) the validity test of the system call number look at the entire `%rax` register. Either would be sufficient to close the current hole, but Anvin's patch will prevent any new paths into the system call entry code from running afoul of this problem in the future. 

The fact that the old exploit was useful implies that someone could have written a test case in 2007 that might have detected the reintroduction of the problem. A suite of such regression tests, run regularly against the mainline, would be quite useful as a way to reduce regressions, both for normal bugs as well as for security holes. Not all kernel bugs will be amenable to that kind of testing, but, for those that are, it seems like an idea worth pursuing. 

The other problem that Hawkes found ([CVE-2010-3081](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3081>), also just reserved) is that the `compat_alloc_user_space()` function did not check to see that the pointer which is being returned is actually a valid user-space pointer. That routine is used to allocate some stack space for massaging 32-bit data into its 64-bit equivalent before making a system call. Hawkes found two places (and believes there are others) where the lack of an `access_ok()` call in that path could be exploited to allow attackers to write to kernel memory. 

One of those was in a video4linux `ioctl()`, but the more easily exploited spot was in the IP multicast `getsockopt()` call. It uses a 32-bit unsigned length parameter provided by user space that can be used to confuse `compat_alloc_user_space()` into returning a pointer into kernel memory. The `compat_mc_getsockopt()` call then writes user-supplied values using those pointers. That can be fairly easily turned into an exploit as Hawkes noted: 

This path allows an attacker to write a chosen value to anywhere within the top 31 bits of the kernel address space. In practice, this seems to be more than enough for exploitation. My proof of concept overwrote the interrupt descriptor table, but it's likely there are other good options too. 

Anvin [patched](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=c41d68a513c71e35a14f66d71782d27a79a81ea6>) `compat_alloc_user_space()` so that it always does the `access_ok()` check. That should take care of the two problem spots that Hawkes found as well as any others that are lurking out there. But there have been a whole lot of kernels released with one or both of these bugs, and there have been other bugs associated with 64-bit/32-bit compatibility. It is a part of the kernel that Hawkes calls ""a _little bit_ scary"": 

Not just because it's an increased attack surface versus having purely 32-bit or purely 64-bit modes, but because of the type of input processing that has to be performed by any such compatibility layer. It invariably involves a significant amount of subtle bit wrangling between 32/64-bit values, using primitives that I'd argue most programmers aren't normally exposed to. The possibility of misuse and abuse is very real. 

Perhaps 32-bit compatibility for x86_64 kernels would be a good starting point for regression testing. Some enterprise distributions were not affected by CVE-2010-3301 because of the ancient kernels (like RHEL's 2.6.18) they are based on, but CVE-2010-3081 was [backported into RHEL 5](<https://access.redhat.com/kb/docs/DOC-40265>), which required that kernel to be updated. The interests of distribution vendors would be well-served by better--any--regression testing so a project of that sort would be quite welcome. The vendors may already be running some tests internally, but regression testing is just the kind of project that would benefit from some cross-distribution collaboration. 

It should also be noted that a [posting](<http://seclists.org/fulldisclosure/2010/Sep/268>) to the full-disclosure mailing list claims that the vulnerability in `compat_mc_getsockopt()` has been known for nearly two-and-a-half years by black (or at least gray) hats. According to the post, it was noticed when the vulnerability was [introduced](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=dae50295488f35d2d617b08a5fae43154c947eec>) in April 2008. Certainly there are some that are following the commit-stream to try to find these kinds of vulnerabilities; it would be good if the kernel had a team of white hats doing the same.
