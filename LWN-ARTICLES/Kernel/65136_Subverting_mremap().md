---
title: Subverting mremap()
url: https://lwn.net/Articles/65136/
date: "January 6, 2004"
category: "Security-Vulnerabilities"
author: 
---

The `mremap()` system call allows a user process to make changes to an existing memory mapping. This call, as exported by the C library, allows changing the size of a mapped region. The underlying call provided by the kernel, however, has an extra parameter which can be used to request that the entire region be moved to a different virtual address. That capability is rarely used, but it turns out to be the key to a new kernel exploit. 

The code implementing `mremap()` makes several checks to ensure that the calling process is not trying to do anything overly strange. The kernel developers forgot to check, however, whether the user has asked to remap a zero-length memory region. In that case, the code does the wrong thing, and creates a new memory area with a length of zero at the requested address. Since numerous places in the virtual memory subsystem code assume that zero-length VM areas do not exist, the creation of such an area is, in effect, a corruption of the kernel's virtual memory data structures. 

The existence of a zero-length virtual memory area is not necessarily a problem; since it does not actually cover any memory, it cannot be used directly to access a memory range which should be off-limits to the process. Where things go wrong is when the kernel makes a pass over a process's entire virtual address space. For example, the `fork()` system call must copy the process's memory space. The code used implements (in a complicated way) a `do` loop that assumes each virtual memory area contains at least one page. As a result, it copies page table information which does not actually exist. 

The situation is complicated by the fact that `mremap()` is happy to create this zero-length area just above the end of the virtual address range allocated to user space--at the beginning of kernel space, in other words. When `fork()` tries to copy the page table information for that area, it can get tangled up in the special large page table entries used for the kernel. The result is a mess. 

What will usually happen (as people who have tried an exploit posted on Bugtraq have found out) is that the system panics and reboots. It is not clear to many people who have looked at the problem ([including Linus](<https://lwn.net/Articles/65160/>)) that this bug can be exploited for anything other than a denial of service attack. It is worth noting, however, that [the advisory](<http://isec.pl/vulnerabilities/isec-0012-mremap.txt>) posted by Paul Starzetz claims: 

Proper exploitation of this vulnerability may lead to local privilege escalation including execution of arbitrary code with kernel level access. Proof-of-concept exploit code has been created and successfully tested giving UID 0 shell on vulnerable systems.... We have identified at least two different attack vectors for the 2.4 kernel series. 

It would not be a good idea to wait and see whether these claims are borne out or not. Prudent administrators will upgrade to the 2.4.24 kernel, or apply the update provided by their distributor. (The 2.6.0 kernel is also vulnerable; the fix can be found in the 2.6.1-rc2 release).
