---
title: The rest of the 5.0 merge window
url: https://lwn.net/Articles/776034/
date: "January 7, 2019"
category: "Releases-5.0"
author: "By Jonathan Corbet January 7, 2019"
---

By **Jonathan Corbet**  
January 7, 2019

Linus Torvalds [released 5.0-rc1](<https://lwn.net/Articles/776102/>) on January 6, closing the merge window for this development cycle and confirming that the next release will indeed be called "5.0". At that point, 10,843 non-merge change sets had been pulled into the mainline, about 2,100 since [last week's summary](<https://lwn.net/Articles/775698/>) was written. Those 2,100 patches included a number of significant changes, though, including some new system-call semantics that may yet prove to create problems for existing user-space code. 

The most significant changes merged in the last week include: 

#### Architecture-specific

  * The C-SKY architecture has gained support for CPU hotplugging, ftrace, and perf. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Core kernel

  * There is a new "dynamic events" interface to the tracing subsystem. It unifies the three distinct interfaces (for kprobes, uprobes, and synthetic events) into a single control file. See [this patch posting](<https://lwn.net/ml/linux-kernel/154140838606.17322.15294184388075458777.stgit@devbox/>) for a brief overview of how this interface works. 

#### Hardware support

  * **Miscellaneous** : NVIDIA Tegra20 external memory controllers, Qualcomm PM8916 watchdog timers, TQ-Systems TQMX86 watchdog timers, MediaTek Command-Queue DMA controllers, UniPhier MIO DMA controllers, Raspberry Pi touchscreens, Amlogic Meson PCIe host controllers, and Socionext UniPhier PCIe controllers. 
  * **Pin control** : NXP IMX8QXP pin controllers, Mediatek MT6797 and MT7629 pin controllers, Actions Semi S700 pin controllers, and Renesas RZ/A2 GPIO and pin controllers. 
  * Support for high-resolution mouse scroll wheels has been significantly improved. 

#### Security

  * A small piece of the secure-boot lockdown patch set has landed in the form of additional control over the `kexec_load_file()` system call. There is a new keyring (called `.platform`) for keys provided by the platform; it cannot be updated by a running system. Keys in this ring can be used to control which images may be run via `kexec_load_file()`. It has also become possible for security modules to prevent calls to `kexec_load()`, which cannot be verified in the same manner. 
  * The secure computing (seccomp) mechanism [can now defer policy decisions to user space](<https://lwn.net/Articles/756233/>). See [this new documentation](<https://lwn.net/Articles/776035/#userspace>) for details on the final version of the API. 
  * The fscrypt filesystem encryption subsystem has gained support for the Adiantum encryption mode (which was added earlier in the merge window). 
  * The semantics of the [`mincore()`](<http://man7.org/linux/man-pages/man2/mincore.2.html>) system call have changed; see below for details. 

#### Internal kernel

  * The venerable `access_ok()` function, which verifies that an address lies within the user-space region, has lost its first argument. This argument was either `VERIFY_READ` or `VERIFY_WRITE` depending on the type of access, but no implementation of `access_ok()` actually used that information. The new prototype is: 

```
int access_ok(void *address, int len);
```

[The patch implementing this change](<https://git.kernel.org/linus/96d4f267e40f9509e8a66e2b39e8b95655617693>) ended up modifying over 600 files. There have also been several follow-up patches fixing various issues created by this change. 

#### Changing mincore()

The `mincore()` system call is used to determine which pages in a virtual address-space range are currently resident in the page cache; the idea is to allow an application to learn which of its pages can be accessed without incurring page faults. As Torvalds notes in [this commit](<https://git.kernel.org/linus/574823bfab82d9d8fa47f422778043fbb4b4f50e>), the intended semantics of this call have always been ""somewhat unclear"", but its behavior all along has been to indicate which pages are resident in the cache, regardless of whether the calling process has ever tried to access those pages. In other words, `mincore()` would reveal the presence of pages faulted in by other processes running in the system. 

Naturally, it turns out that if you can observe aspects of the system state that are the result of other process's activity, you can use that information to extract information that should be hidden. Daniel Gruss et al. have recently [released a paper [PDF]](<https://arxiv.org/pdf/1901.01161.pdf>) showing how `mincore()` can be exploited in just this manner. In response, Jiri Kosina posted [a patch](<https://lwn.net/ml/linux-kernel/nycvar.YFH.7.76.1901051817390.16954@cbobk.fhfr.pm/>) allowing system administrators to turn `mincore()` into a privileged system call by way of a sysctl knob, but Torvalds [wasn't pleased](<https://lwn.net/ml/linux-kernel/CAHk-=wicks2BEwm1BhdvEj_P3yawmvQuG3NOnjhdrUDEtTGizw@mail.gmail.com/>) with that approach. He responded with [a patch](<https://lwn.net/ml/linux-kernel/CAHk-=wif_9nvNHJiyxHzJ80_WUb0P7CXNBvXkjZz-r1u0ozp7g@mail.gmail.com/>) restricting the information returned by `mincore()` to anonymous pages and a small subset of file pages. 

After Jann Horn [pointed out](<https://lwn.net/ml/linux-kernel/CAG48ez2jAp9xkPXQmVXm0PqNrFGscg9BufQRem2UD8FGX-YzPw@mail.gmail.com/>) that restricting the query to the calling process's page tables reduces the attack surface considerably, though, Torvalds decided to change his approach. As a result, the patch that was committed adds no new knobs, but does unconditionally restrict `mincore()` to pages that are actually mapped by the calling process — pages that said process has accessed at some point. That makes it much harder to use `mincore()` to observe what other processes are doing; as Torvalds [pointed out](<https://lwn.net/ml/linux-kernel/CAHk-=whL4sZiM=JcdQAYQvHm7h7xEtVUh+gYGYhoSk4vi38tXg@mail.gmail.com/>), though, such observation is still theoretically possible, but harder. 

So the easy attack is closed, but that additional security may come at the cost of creating problems for user space. As Torvalds noted in the changelog: 

NOTE! This is a real semantic change, and it is for example known to change the output of "fincore", since that program literally does a mmap without populating it, and then doing "mincore()" on that mapping that doesn't actually have any pages in it. 

I'm hoping that nobody actually has any workflow that cares, and the info leak is real. 

If the change breaks code in the wild, it may have to be reverted and some other solution found; for this reason, this patch has not been marked for inclusion into the stable kernels. For those out there who have code that uses `mincore()`, now would be a good time to test the new semantics to ensure that things still work as expected. 

A couple of significant things were _not_ merged before the merge window closed, including [the controversial fs-verity patch set](<https://lwn.net/Articles/775872/>). Also missing again is [the new filesystem mounting API](<https://lwn.net/Articles/759499/>), though some of the precursor patches did go in toward the end of the merge window. Unless something surprising happens, the feature set for this cycle is complete and the 5.0 kernel is now in the stabilization phase, with a final release expected in late February.
