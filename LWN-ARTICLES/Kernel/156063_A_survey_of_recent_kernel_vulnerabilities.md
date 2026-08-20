---
title: A survey of recent kernel vulnerabilities
url: https://lwn.net/Articles/156063/
date: "October 18, 2005"
category: "Security-Vulnerabilities"
author: 
---

There has been a fairly long list of kernel vulnerabilities over the last few months, but few of them have received much serious attention (outside of the security groups at numerous distributors, who have been duly issuing patches as the issues come up). Here's a selection of recent problems. 

CVE| Fixed-in| Description  
---|---|---  
[CAN-2005-2098](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-2098>) | 2.6.12.5  
2.6.13 | The session keyring code had an error path which could fail to release the session management semaphore. As a result, any local user could cause processes to hang.  
[CAN-2005-2099](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-2099>) | 2.6.12.5  
2.6.13 | A keyring which failed to instantiate correctly could leave behind a NULL pointer which would subsequently be dereferenced by the kernel, causing an oops.  
[CAN-2005-1761](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-1761>) | 2.6.12.1 | A `ptrace()` bug on the ia64 architecture enables local denial of service attacks. ([Patch](<https://lwn.net/Articles/156123/>))  
[CAN-2005-1913](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-1913>) | 2.6.12.1 | The subthread exec code did not properly reparent timers, leading to an oops caused by a local user when signals are delivered to the wrong thread. ([Patch](<https://lwn.net/Articles/156122/>))  
[CAN-2005-2456](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-2456>) | 2.6.13 | The XFRM policy parser had an array overflow, enabling denial of service attacks by local users. ([Patch](<http://www.mail-archive.com/netdev@vger.kernel.org/msg00520.html>))  
[CAN-2005-2457](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-2457>) | 2.6.13 | Mounting a malicious compressed ISO filesystem could lead to a kernel oops  
[CAN-2005-2458](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-2458>)  
[CAN-2005-2459](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-2459>) | 2.6.13 | Two zlib vulnerabilities which can be used to oops the kernel and create denial of service attacks.  
[CAN-2005-2490](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-2490>) | 2.6.13.1 | A race condition with user space allows a local attacker to change the contents of a message passed to the 32-bit version of `sendmsg()` on 64-bit architectures. The result is a locally exploitable buffer overflow. ([Patch](<https://lwn.net/Articles/156070/>))  
[CAN-2005-2492](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-2492>) | 2.6.13.1 | An unchecked user-space dereference in `sendmsg()` can be exploited to oops the system. ([Patch](<https://lwn.net/Articles/156071/>))  
[CAN-2005-2548](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-2548>) | 2.6.9 | A hostile UDP packet could cause the 8021Q VLAN code to oops, leading to remote denial of service attacks.   
[CAN-2005-2555](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-2555>) | 2.6.13 | The kernel failed to restrict kernel socket policy loading to administrative users. ([Patch](<http://www.kernel.org/git/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=6fc0b4a7a73a81e74d0004732df358f4f9975be2>))   
[CAN-2005-3044](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-3044>) | 2.6.13.2 | The 32-bit `ioctl()` handler on x86-64 was missing an `fput()` call. This error could be exploited by a local attacker to corrupt kernel data structures. ([Patch](<https://lwn.net/Articles/156121/>))  
[CAN-2005-3053](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-3053>) | 2.6.13 | The `set_mempolicy()` system call, used to tweak memory behavior on NUMA systems, did not properly check the `policy` argument. A local attacker could, by supplying a negative value, could cause a kernel oops. ([Patch](<http://www.kernel.org/git/?p=linux/kernel/git/torvalds/linux-2.6.git;a=blobdiff;h=b4eababc8198790961ead62cf421b2ac2c21aaf5;hp=1694845526be5db9c18ef0183110435a0ccf9ab7;hb=ba17101b41977f124948e0a7797fdcbb59e19f3e;f=mm/mempolicy.c>))  
[CAN-2005-3106](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-3106>) | 2.6.11 | A race condition between core dumps and `exec()` could enable a local attacker to deadlock the system. ([Patch](<http://linux.bkbits.net:8080/linux-2.6/diffs/fs/exec.c@1.156?nav=index.html|src/|src/fs|hist/fs/exec.c>))  
[CAN-2005-3107](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-3107>) | 2.6.11 | Another local deadlock related to core dumps and `ptrace()`. ([Patch](<http://www.kernel.org/pub/linux/kernel/people/akpm/patches/2.6/2.6.11-rc1/2.6.11-rc1-mm1/broken-out/fix-coredump_wait-deadlock-with-ptracer-tracee-on-shared-mm.patch>))  
[CAN-2005-3108](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-3108>) | 2.6.11 | The right sort of I/O mapping could create information leaks and kernel oopses on the x86-64 platform. It is hard to see how this one could be exploited by an unprivileged user. ([Patch](<http://www.kernel.org/git/?p=linux/kernel/git/torvalds/linux-2.6.git;a=blobdiff;h=c6fb0cb69992bbf14a2f42d4ddde10b298cd9316;hp=74ec8554b195de6c5a9b87ce5d39f08d9c5da544;hb=93ef70a217637ade3f335303a112b22a134a1ec2;f=arch/x86_64/mm/ioremap.c>))  
[CAN-2005-3109](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-3109>) | 2.6.11 | A maliciously created HFS filesystem could oops the kernel, if the system was configured to allow users to mount such filesystems. ([Patch](<https://lwn.net/Articles/156065/>))  
[CAN-2005-3110](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-3110>) | 2.6.12 | A race condition in the netfilter ebtables module can cause a kernel oops on SMP systems. ([Patch](<http://sourceforge.net/mailarchive/forum.php?thread_id=6800453&forum_id=8572>)).  
[CAN-2005-3119](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-3119>) | 2.6.13.4 | A memory leak in the key request code could be used in denial of service attacks. ([Patch](<https://lwn.net/Articles/156124/>))  
[CAN-2005-3180](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-3180>) | 2.6.13.4 | The orinoco driver can leak information onto the net. ([Patch](<https://lwn.net/Articles/156125/>))  
[CAN-2005-3181](<http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CAN-2005-3181>) | 2.6.13.4 | A memory leak in the audit code can be used for denial of service attacks. ([Patch](<https://lwn.net/Articles/156126/>))  
  
That is a long list of vulnerabilities. The fact that almost all of them are "only" denial of service problems, and that only one of those is truly remotely exploitable, is of limited consolation. 

One may well wonder why the kernel is the source of so many security holes, far more than any other package on the system. The complexity of the kernel and the environment in which it runs, the fact that many often-harmless bugs (such as memory leaks) turn into security issues for the kernel, and the high level of auditing which is done on kernel code are all part of the answer to that question. Unfortunately, the flow of security issues in the kernel is unlikely to stop anytime soon.
