---
title: "Kernel lockdown in 4.17?"
url: https://lwn.net/Articles/750730/
date: "April 2, 2018"
category: "Security-UEFI secure boot"
author: "By Jonathan Corbet April 2, 2018"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 2, 2018

The UEFI secure boot mechanism is intended to protect the system against persistent malware threats — unpleasant bits of software attached to the operating system or bootloader that will survive a reboot. While Linux has supported secure boot for some time, proponents have long said that this support is incomplete in that it is still possible for the root user to corrupt the system in a number of ways. Patches that attempt to close this hole have been circulating for years, but they have been controversial at best. This story may finally come to a close, though, if Linus Torvalds accepts the "kernel lockdown" patch series during the 4.17 merge window. 

In theory, the secure-boot chain of trust ensures that the system will never run untrusted code in kernel mode. On current Linux systems, though, the root user (or any other user with sufficient capabilities) can do exactly that. For anybody who wants to use secure boot to ensure the integrity of their systems (or, perhaps, to prevent their customers from truly owning the system), this hole defeats the purpose of the whole exercise. Various kernel lockdown patches have been proposed over the years (LWN first [covered them](<https://lwn.net/Articles/514985/>) in 2012), but these patches have run into two types of criticism: (1) restricting what root can do goes against the design of Unix-like systems, and (2) locking down the system in this way still does not solve the problem. 

Interest in this kind of lockdown persists, though, and the lockdown patches have been quietly maintained and improved over the years. These patches were last publicly [posted](<https://lwn.net/Articles/738692/>) in November. They were [added to linux-next](<https://lwn.net/Articles/750761/>) at the beginning of March, and found their way into the security tree on March 31\. Security maintainer James Morris has not yet pushed the lockdown patches to Torvalds, but has [indicated](<https://lwn.net/Articles/750763/>) his intent to do so in the near future. So, unless Torvalds objects, it would appear that kernel lockdown will be a part of the 4.17 release. 

#### How to lock down the kernel

The series of patches that implements this feature can be seen [on kernel.org](<https://git.kernel.org/pub/scm/linux/kernel/git/dhowells/linux-fs.git/log/?h=lockdown-20180330>). It starts with [a patch](<https://git.kernel.org/pub/scm/linux/kernel/git/dhowells/linux-fs.git/commit/?h=lockdown-20180330&id=1235d72fe1d34f9961051d159af3b48a1617ff0a>) adding a configuration option for the lockdown feature and a `kernel_is_locked_down()` function to test whether lockdown is in effect. There is a `lockdown` boot parameter that can be used to enable lockdown, but lockdown also happens automatically if the feature is enabled and UEFI secure boot is detected. After this patch, the mode is implemented but it doesn't actually lock anything down yet. 

No sooner was the lockdown option added than somebody wanted to be able to turn it off. Normally, the ability to defeat this kind of protection would be considered undesirable. In this case, it can only be done from the system console with the magic `SysRq-x` key sequence. 

After that, the real restrictions are added. In lockdown mode, the kernel will refuse to load modules unless they either have a recognized signature or they can pass appraisal from the integrity measurement architecture (IMA) subsystem. Access to `/dev/mem` and `/dev/kmem` (which is already heavily restricted on most systems) is disabled entirely. `/proc/kcore`, which can be used to pull cryptographic keys from the kernel, is disabled. 

Access to I/O ports via `/dev/port` is disallowed lest it be used to convince a device to overwrite important memory via a DMA operation. There are other places where hardware might be used to circumvent lockdown. To try to prevent that, any sort of access to PCI bus operations or parameters (the location of the base address register, for example) must be disabled. The [`ioperm()`](<http://man7.org/linux/man-pages/man2/ioperm.2.html>) and [`iopl()`](<http://man7.org/linux/man-pages/man2/iopl.2.html>) system calls are disabled, as is direct access to x86 model-specific registers (MSRs). 

The [ACPI custom method mechanism](<https://www.kernel.org/doc/Documentation/acpi/method-customizing.txt>) is also capable of overwriting memory, so it is disabled. There are a few other hazards in ACPI, including the `acpi_rsdp` command-line parameter, which can be used to change the [RSDP](<https://wiki.osdev.org/RSDP>) structure. The [ACPI table override](<https://www.kernel.org/doc/Documentation/acpi/initrd_table_override.txt>) and [APEI error injection](<https://www.kernel.org/doc/Documentation/acpi/apei/einj.txt>) mechanisms are turned off as well. 

None of these changes will help in the case where a specific device driver can be used to overwrite memory. For example, the "eata" SCSI driver has a set of command-line options that control many of its hardware-level parameters; those must be disabled when lockdown is in effect. If anybody is trying to lock down a system that has PCMCIA hardware (younger readers can ask their parents about this), the ability to replace the card information structure (CIS) must be disabled. The serial-port `TIOCSSERIAL` `ioctl()` operation is problematic, since it can change I/O port settings. Many other hardware-controlling command-line parameters were [annotated](<https://lwn.net/Articles/708179/>) in 2016; under kernel lockdown they become unavailable. 

There are a number of restrictions on the kexec mechanism (used to boot a new kernel directly from a running kernel). If the root user can switch to an arbitrary kernel, the system is clearly not locked down, so that must be prevented. The [`kexec_load()`](<http://man7.org/linux/man-pages/man2/kexec_load.2.html>) system call is disabled entirely; `kexec_file_load()`, which [can enforce signatures](<https://lwn.net/Articles/603116/>), is still allowed. A mechanism has been added to ensure that the newly loaded kernel retains the secure-boot mode. Once again, IMA can be used as a substitute for signature verification. 

"Hibernation" is the process of saving a snapshot of system memory to disk and powering down the system, with the intent of resuming from that snapshot in the future. There is also a variant called "uswsusp" where most of the work is done in user space. In either case, the kernel currently lacks a mechanism to ensure that the image it resumes from is the same as the one it saved at hibernation time. In other words, some clever editing of the hibernation image could be used to bypass lockdown. That is addressed by disabling hibernation and uswsusp entirely. 

This whole exercise will be in vain, though, if an attacker is able to install kprobes into the running kernel, so that feature, too, is turned off. Some BPF operations that can read memory are also disabled. These changes will adversely affect a number of tracing use cases. The debugfs filesystem is disabled entirely on locked-down systems. 

#### A secure system?

The end result of all this work is intended to be a kernel that cannot be subverted even by a privileged user. As is often the case with such features, kernel lockdown is a two-edged sword. A cloud-hosting provider has a strong incentive to ensure that its operating software has not been compromised, but that is also true of a provider of Linux-based consumer-electronics gadgets that simply wants to preserve a rent-seeking business model. It is hard to protect one type of system without protecting the other. 

It is also hard to imagine a scenario where the lockdown feature provides complete protection. If nothing else, there must be a lot of device drivers that can still be used to get around restrictions. But proponents of the lockdown feature will argue that it raises the bar considerably, and that is generally the best that can be hoped for in this world. The one bar that still has to be overcome, though, is acceptance into the mainline. Torvalds has not yet expressed an opinion on this patch set. It would seem that much of the opposition to its merging has been overcome (or worn down), though, so merging into the mainline for 4.17 seems probable.
