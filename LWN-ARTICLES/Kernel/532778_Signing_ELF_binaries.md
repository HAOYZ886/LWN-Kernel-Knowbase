---
title: Signing ELF binaries
url: https://lwn.net/Articles/532778/
date: "January 16, 2013"
category: "Security-UEFI secure boot"
author: "By Jake Edge January 16, 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
January 16, 2013

As part of the effort to support UEFI secure boot on Linux, Matthew Garrett proposed [a number of restrictions](<https://lwn.net/Articles/514985/>) on kernel features so that signed kernels could not be used to circumvent secure boot. Many of those restrictions were fairly uncontroversial, but disabling `kexec()` was not one of them, so it was [dropped in a later patch set](<https://lwn.net/Articles/523367/>). At the time, there was discussion of how to support `kexec()` in a secure boot world; Vivek Goyal recently posted an [RFC patch set](<https://lwn.net/Articles/532710/>) to start down that path. 

The `kexec()` system call is used to replace the running kernel with a different program. It can be used to boot a new kernel without going through the BIOS or other firmware, which is exactly what gets it into trouble for secure boot. A running kernel that has been verified by the secure boot mechanism (and thus is trusted) could boot any unsigned, unverified kernel by way of `kexec()`. The concern is that it would be used to boot Windows in an insecure environment while making it believe it was running under secure boot--exactly what secure boot is meant to prevent. That, in turn, could lead to Linux bootloaders getting blacklisted, which would make it more difficult to boot Linux on hardware certified for Windows 8. 

Goyal's patches add the ability to cryptographically sign ELF executables, then have the kernel verify those signatures. If the binary is signed and the signature verifies, it will be executed. While the patch does not yet implement this, the idea is that a signed binary could be given additional capabilities if it verifies--capabilities that would enable `kexec()`, for example. If the binary is unsigned, it will always be executed. Only if a signed binary fails to verify does it get blocked from execution. 

The patches contain a `signelf` utility that puts a signature based on the private key argument into a `.signature` ELF section. The signature is calculated by hashing the contents of the PT_LOAD ELF segments, then cryptographically signing the result. It is based on the [module signing code](<https://lwn.net/Articles/525592/>) that was recently added to the kernel, but instead of just tacking the signature on at the end of the binary, it puts it into the `.signature` section. 

Since any shared libraries used by an executable cannot be trusted (so far, at least, there is no mechanism to verify those libraries), only statically linked executables can be signed and verified. The patches do not stop binaries from using `dlopen()` directly, however, so Goyal said binaries that do so should not be signed. He is targeting the `/sbin/kexec` binary that is used to launch `kdump`, so that users can still get crash dumps, even in a secure-boot-enabled system, but there are other possible uses as well. 

When the `binfmt_elf` loader in the kernel detects a binary with the `.signature` section, it locks the pages of the executable into memory and verifies the signature. Goyal is trying to avoid situations where the binary is modified after the verification has been done, which is why the executable is locked into memory. If the signature does not verify, the process is killed; unsigned binaries are simply executed as usual. 

Beyond just adding the capability for `kexec()`, there are some other pieces of the puzzle that aren't addressed in the patches. The biggest is the need to disable `ptrace()` on signed binaries. Otherwise, the signed binary could be subverted in various ways--changing the binary passed to `kexec()`, for example. In addition, the "to do" list has some key and keyring related issues that need to be sorted out. 

There is already a mechanism in the kernel to verify the signature of various kinds of files, though. The [Integrity Measurement Architecture (IMA) appraisal extension](<https://lwn.net/Articles/488906/>) that was added in Linux 3.7 does much of what Goyal needs, as was [pointed out](<https://lwn.net/Articles/532842/>) by IMA maintainer Mimi Zohar. While the integrity subsystem targets measuring and verifying the whole system, it already does most of the kinds of signature operations Goyal is looking to add. On the other hand, features like disabling `ptrace()`, locking the binary into memory, and setting capabilities based on signature verification are well beyond the scope of the integrity subsystem. Goyal is currently looking into using the integrity features and adding secure-boot-specific features on top. 

Losing the ability to use `kexec()` on secure boot systems would be rather painful. While Garrett's patches do not actually make that change (because of the outcry from other kernel developers), any distribution that is trying to enable secure boot is likely to do so. Finding a way to support that use case, without unduly risking the blacklist wrath of Microsoft, would be good.
