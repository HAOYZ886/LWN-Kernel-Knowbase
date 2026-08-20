---
title: A kernel integrity subsystem update
url: https://lwn.net/Articles/753276/
date: "May 2, 2018"
category: Integrity measurement architecture
author: "By Jake Edge May 2, 2018 LSFMM"
---

By **Jake Edge**  
May 2, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

At the 2018 Linux Storage, Filesystem, and Memory-Management Summit, Mimi Zohar gave a presentation in the filesystem track on the Linux integrity subsystem. There is a lot of talk that the integrity subsystem (usually referred to as "IMA", which is the [integrity measurement architecture](<https://sourceforge.net/p/linux-ima/wiki/Home/>), though there is more to the subsystem) is complex and not documented well, she said. So she wanted to give an overview of the subsystem and then to discuss some filesystem-related concerns. 

The goals of the integrity subsystem are to detect files that have been altered, either accidentally or maliciously, by comparing a measurement of the current file contents with that of a stored "good" value; it can then enforce various file integrity policies (e.g. no access or no execution). IMA is three separate pieces: measurement, which calculates a hash of file contents; appraisal, which verifies file signatures made using the measured hashes; and audit, which records hashes and other information in the audit logs. There is also the extended verification module (EVM), which targets the measurement and protection of the file metadata, though that is not included in what she would be presenting. 

It is important to note that IMA does not protect against attacks on objects in memory, it can only be used to thwart attacks that change files. The policies governing IMA behavior for a given system all come from a single file. There are two built-in policies, one that uses the Trusted Platform Module (TPM) to sign a list of file hashes that can be used for attestation to a third party. The other can verify the entire [trusted computing base](<https://en.wikipedia.org/wiki/Trusted_computing_base>) (TCB) of the system. The IMA policy file can itself be signed and verified, of course. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

One reason why IMA is seen as complicated and difficult to work with is because there are so many pieces to it. One area that needs work is software distribution with signature verification. RPM has provisions for signatures on package files, but there have been three separate attempts to add signatures to `.deb` files, without success. Key management is another area; there needs to be a separation between the keys that are used before starting the operating system and those that are trusted after that point. 

IMA-audit was added in the 3.10 kernel; it can be used to augment existing security information and event management (SIEM) tools, as FireEye [described in 2016](<https://www.fireeye.com/blog/threat-research/2016/11/extending_linux_exec.html>). There is a problem, though: it is important to be able to identify the namespace or container that is generating the log entry. There is a [proposal](<https://lwn.net/Articles/750313/>) for a kind of container ID to be used by the audit subsystem, but nothing has been merged as yet. 

Most of the needed hooks to measure files and verify signatures are available. One missing piece is the ability to verify the signature on the root filesystem, which requires a `cpio` that can handle security extended attributes (xattrs) on files in the initramfs. Some patches to add that functionality to `cpio` have been posted, but she wonders if there is a maintainer of the tool that will pick up and maintain that work. 

All code and data that gets loaded into the kernel needs to be able to be measured and have signatures verified. That includes kernel modules, policy files, and so on. New system calls that take file descriptor parameters allow IMA to access the needed security xattrs. But she is concerned that BPF programs may avoid measurement and verification since file descriptors are not part of the API for loading them. 

There are still some kinds of attacks that are not being thwarted; the most major is that file names can be changed in an offline attack. Protecting against that would require hashing and verifying the directory structure. Symbolic and hard links would both provide ways to have different names for the same file, however, as was pointed out by several in the room. It is the reason that SELinux protects objects and not files by name, Ted Ts'o noted. 

Namespace support for IMA is another area that needs attention, Zohar said. IMA-measurement needs per-namespace policies and IMA-appraisal needs per-namespace keyrings. She concluded her talk by asking the assembled developers to help ensure that new features did not add measurement or appraisal gaps. There are some difficult problems that need solving for IMA and she would like some help from the filesystem developers in doing so.
