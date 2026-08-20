---
title: Reworking kexec for signatures
url: https://lwn.net/Articles/603116/
date: "June 25, 2014"
category: "Kexec; Security-Security technologies"
author: "By Jake Edge June 25, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
June 25, 2014

The kernel execution (kexec) subsystem allows a running kernel to switch to a different kernel. This allows for faster booting, as the system firmware and bootloader are bypassed, but it can also be used to produce crash dumps using [Kdump](<https://www.kernel.org/doc/Documentation/kdump/kdump.txt>). However, as Matthew Garret explained on his blog, kexec could be used to [circumvent UEFI secure boot restrictions](<http://mjg59.dreamwidth.org/28746.html>), which led him to [propose](<https://lwn.net/Articles/514985/>) a way to disable kexec on secure boot systems. That was not terribly popular, but a more recent patch set would provide a path for kexec to only boot _signed_ kernels, which would solve the problem Garrett was trying to address, without completely disabling the facility. 

The kexec subsystem consists of the [`kexec_load()`](<http://man7.org/linux/man-pages/man2/kexec_load.2.html>) system call that loads a new kernel into memory, which can then be booted using the [`reboot()`](<http://man7.org/linux/man-pages/man2/reboot.2.html>) system call. There is also a [`kexec` command](<http://linux.die.net/man/8/kexec>) that will both load the new kernel and boot it, without entering the system firmware (e.g. BIOS or UEFI) and bootloader. 

But the UEFI firmware is what enforces the secure boot restrictions. Garrett was concerned that a Linux kernel could be used to boot an unsigned (and malicious) Windows operating system by way of kexec because it circumvents secure boot. That might lead Microsoft to blacklist the keys used to sign _Linux_ bootloaders, which would make it difficult to boot Linux on commodity hardware. Using kexec that way could affect secure-booted Linux systems too, of course, though Microsoft might not be so quick to revoke keys under those circumstances. 

In any case, Garrett eventually [removed](<https://lwn.net/Articles/523367/>) the kexec-disabling portion of his patch set (though he strongly suggested that distributions should still disable kexec if they are going to support secure boot). Those patches have not been merged (yet?). More recently, Vivek Goyal has put together a [patch set](<https://lwn.net/Articles/601213/>) that is intended to address Garrett's secure boot concerns, but would also protect systems that only allow loading signed kernel modules. As Garrett showed in his blog post, that restriction can be trivially bypassed by executing a new kernel that simply alters the `sig_enforce` sysfs parameter in the original kernel's memory and then jumps back to that original kernel. 

Goyal's patches start down the path toward being able to restrict kexec so that it will only load signed code. To that end, this patch set defines a new system call: 

```
long kexec_file_load(int kernel_fd, int initrd_fd,
    			 const char *cmdline_ptr, unsigned long cmdline_len,
                             unsigned long flags);
```

It will load the kernel executable from the `kernel_fd` file descriptor and will associate the "initial ramdisk" (initrd) from the `initrd_fd` descriptor. It will also associate the kernel command line passed as `cmdline_ptr` and `cmdline_len`. The initrd and command-line information will be used when the kernel is actually booted. This contrasts with the existing kexec system call: 

```
long kexec_load(unsigned long entry, unsigned long nr_segments,
                        struct kexec_segment *segments, unsigned long flags);
```

It expects to get `segments` that have been parsed out of a kernel binary in user space and to just blindly load them into memory. As can be seen, `kexec_file_load()` puts the kernel in the loop so that it can (eventually) verify what is being loaded and executed. 

As one of the segments that get loaded, there is a standalone executable object, called "purgatory", that runs between the two kernels. At `reboot()` time, the "exiting" kernel jumps to the purgatory code. Its main function is to check the SHA-256 hashes of the other segments that were loaded. If those have not been corrupted, booting can proceed. The purgatory code will copy some memory to a backup region and do some architecture-specific setup, then jump to the new kernel. 

The purgatory code currently lives in [kexec-tools](<https://www.kernel.org/pub/linux/utils/kernel/kexec/>), but if the kernel is to take responsibility for setting up the segments from the kernel binary and initrd, it will need a purgatory of its own. Goyal's patch set [adds that code](<https://lwn.net/Articles/603414/>) for x86 to `arch/x86/purgatory/`. 

Goyal also [copied code from `crypto/sha256_generic.c`](<https://lwn.net/Articles/603417/>) into the purgatory directory. It's clear he would rather simply just use the code directly from the `crypto/` directory, but could not find a way to do so: 

I could not #include sha256_generic.c file here as some of the function signature [required a] little tweaking. Original functions work with crypto API but these ones don't 

So instead of doing #include on sha256_generic.c I just copied relevant portions of code into arch/x86/purgatory/sha256.c. Now we shouldn't have to touch this code at all. Do let me know if there are better ways to handle it. 

While the patch set is at version 3 (earlier versions: [v2](<https://lwn.net/Articles/582711/>), [v1](<https://lwn.net/Articles/574400/>)), it is still a "request for comment" (RFC) patch. There are various unfinished pieces, with signature verification topping the list. So far, the new facility is only available for the x86_64 architecture and bzImage kernel images. Adding other architectures and support for the ELF kernel format still remain to be done. There is also a need for some documentation, including a `man` page. 

Goyal did [explain](<https://lwn.net/Articles/603428/>) his vision for how the signature verification will work. It is based on David Howells's work on verifying the signatures for loadable kernel modules. Essentially, the signature will be verified when `kexec_load_file()` is called. That is also when the SHA-256 hashes for each segment are calculated and stored in the purgatory segment. So, all purgatory has to do is verify the hashes (which it already does to avoid running corrupted code) to ensure that only a properly signed kernel will be executed. 

There have been plenty of comments on each version of the patch set, but most of those on v3 were technical suggestions for improving the code. So far, there have been no complaints about the overall idea, which means we may well see the ability to require cryptographic signatures on the kernels passed to kexec added as a feature sometime in the next year—hopefully sooner than that. It would be a nice feature to have when Garrett's secure boot patches get merged.
