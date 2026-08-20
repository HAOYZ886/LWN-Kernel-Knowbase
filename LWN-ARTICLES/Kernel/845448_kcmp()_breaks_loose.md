---
title: kcmp() breaks loose
url: https://lwn.net/Articles/845448/
date: "February 11, 2021"
category: "System calls-kcmp"
author: "By Jonathan Corbet February 11, 2021"
---

By **Jonathan Corbet**  
February 11, 2021

Given the large set of system calls implemented by the Linux kernel, it would not be surprising for most people to be unfamiliar with a few of them. Not everybody needs to know the details of [`setresgid()`](<https://man7.org/linux/man-pages/man2/setresuid.2.html>), [`modify_ldt()`](<https://man7.org/linux/man-pages/man2/modify_ldt.2.html>), or [`lookup_dcookie()`](<https://man7.org/linux/man-pages/man2/lookup_dcookie.2.html>), after all. But even developers who have a wide understanding of the Linux system-call set may be surprised by [`kcmp()`](<https://man7.org/linux/man-pages/man2/kcmp.2.html>), which is not enabled by default in the kernel build. It would seem, though, that the word has gotten out, leading to an effort to make `kcmp()` more widely available. 

The `kcmp()` system call was [added in 2012](<https://lwn.net/Articles/478111/>) to address a problem encountered by the [checkpoint/restore in user space](<https://criu.org/Main_Page>) (CRIU) effort. The CRIU developers are working (with some success) toward the goal of being able to record the complete state of a set of processes to persistent storage, then restart those processes at some future time, possibly on a different machine. This would be challenging in the best of times, but the CRIU developers have taken on an additional handicap: doing the entire job from user space. Over the years attempts have been made to implement a kernel-based checkpoint mechanism, but none have even come close to being considered for merging. The user-space approach appears to be the only realistic way of solving the checkpoint/restore problem. 

CRIU may be banished to user space, but the kernel community has still allowed the addition of features where needed to get the job done. For example, [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) helps in the migration of process memory, and various features of the [`clone()`](<https://man7.org/linux/man-pages/man2/clone.2.html>) system call help with recreating processes that look the same as they did at checkpoint time. These helpers have made the checkpoint/restore job doable while still keeping most of the work out of the kernel. 

One problem the CRIU developers encountered early on was determining whether two open file descriptors (possibly found in different processes) refer to the same open file within the kernel. Creating such file descriptors can be done with [`dup()`](<https://man7.org/linux/man-pages/man2/dup.2.html>) or `clone()`; they can be spread across unrelated processes by sending `SCM_RIGHTS` datagrams. It was easy enough for CRIU to determine that two file descriptors refer to the same file by looking at the relevant entries in `/proc`; at restore time, that file can be opened again in both places to recreate the situation at checkpoint time. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

If, however, two file descriptors refer to the same _open_ file — if, in other words, they refer to the same `file` structure within the kernel — then replacing them with two independent descriptors at restore time may break the application. CRIU can do the right thing to restore these descriptors correctly, but only if it can detect that they are related at checkpoint time. That detection was not something that the kernel supported at the time. 

Querying the provenance of file descriptors is, at its core, asking about the kernel's internal data structures; making that information available must be done with great care. One idea that was discussed early on was to have the kernel export the address of the `file` structure behind each descriptor; if two descriptors show the same address, then they are entangled and should be recreated in the same mode. But the kernel goes to some length to hide the addresses of its data structures to make attacks harder; this effort is not always successful, but it is deemed worth doing. So exposing addresses in this way is not something that will fly. 

Instead, the developers finally added a system call to answer the actual question: are these two descriptors the same? That was `kcmp()`: 

```
int kcmp(pid_t pid1, pid_t pid2, int type, unsigned long idx1,
        	     unsigned long idx2);
```

If `type` is `KCMP_FILE`, then the kernel will check whether file descriptor `idx1` in the process whose ID is `pid1` is the same as descriptor `idx2` in `pid2`. There are a number of other resources that can be queried in the same way, but the question is always the same: are these two the same thing? This is a much safer question for the kernel to answer, but there are still restrictions; in particular, the calling process must have the privilege to use [`ptrace()`](<https://man7.org/linux/man-pages/man2/ptrace.2.html>) on both of the target processes, and all processes involved must be in the same PID namespace. 

Even with those restrictions, `kcmp()` made some people nervous. As a way of containing any possible damage, this system call was only built into kernels configured for checkpoint/restore functionality. If it was absent on most kernels, it could not be used to exploit those kernels, after all. 

In the real world, though, the choices made by kernel developers about configuration options mean relatively little. Most users run kernels built by distributors, and distributors have an incentive to enable as many features as possible, even if relatively few users will need them. Most people will not complain about unneeded code in their kernels — code they probably do not even know is there — but they will definitely complain if some feature they need does not work. So, while checkpoint/restore users are relatively rare, distributors (Fedora and Ubuntu, for example) have enabled the feature for those who need it. That has made `kcmp()` widely available as well. 

If you make a feature available, somebody will come along and use it, probably in some way you didn't anticipate. And so, it seems, the [Mesa graphics library](<https://www.mesa3d.org/>) found a use for `kcmp()` that has nothing to do with checkpointing. At times, the library can find itself dealing with multiple file descriptors referring to the same underlying DRM devices; in this case, making changes to one will affect the other, probably in unsatisfying ways. To avoid this problem, Mesa checks (with `kcmp()`) to ensure that file descriptors are independent when needed. 

That check will only work properly if `kcmp()` is actually available in the running kernel, though, and that is not the case on all distributions. Asking those distributors to enable the full checkpoint/restore functionality for `kcmp()` seems like overkill, so Chris Wilson instead [submitted a patch](<https://lwn.net/ml/linux-kernel/20210205163752.11932-1-chris@chris-wilson.co.uk/>) to make `kcmp()` selectable independently. Describing the need for the patch, Daniel Vetter [said](<https://lwn.net/ml/linux-kernel/CAKMK7uHnOA9CuRxcKkcqG8duOw_3dZobkThcV7Q_swMXVoLCkQ@mail.gmail.com/>): 

It was maybe stupid, but our userspace started relying on fd comparison through sys_kcomp. So for better or worse, if you want to run the mesa3d gl/vk stacks, you need this. Was maybe not the brightest idea, but since enough distros had this enabled by defaults, it wasn't really discovered, and now we're shipping this everywhere. 

Michel Dänzer, who implemented this functionality, [defended](<https://lwn.net/ml/linux-kernel/5a940e13-8996-e9e5-251e-a9af294a39ff@daenzer.net/>) the use of `kcmp()` and expressed surprise that it wasn't universally available. He asked what other solution he should have chosen, but got no answer. Kees Cook [noted](<https://lwn.net/ml/linux-kernel/202102051030.1AF01772D@keescook/>) that `kcmp()` ""is a really powerful syscall"", but that its use is constrained and it's already widely available, ""so it may be okay to expose this"". 

The first version of the patch enabled `kcmp()` by default, but that runs counter to normal practice even in the absence of any residual security concerns so, by the [third version](<https://lwn.net/ml/linux-kernel/20210205220012.1983-1-chris@chris-wilson.co.uk/>), the default was changed to "no". The system call will still be enabled, though, if either checkpoint/restore or graphics are enabled, meaning that it will be available on most kernels going forward. It would be fairly surprising if this patch were not merged for 5.12, and distributors may well backport it to older kernels as well.
