---
title: IMA appraisal extension
url: https://lwn.net/Articles/488906/
date: "March 28, 2012"
category: "Integrity measurement architecture; Security-Integrity verification"
author: "By Jake Edge March 28, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
March 28, 2012

The "integrity" of a Linux system is based on whether it is running the code that the administrator expects. If not, a compromise of the system may have occurred. The Linux integrity subsystem is meant to detect those unexpected changes to files in order to protect systems against compromise. That is done by creating integrity "measurements" (hashes of contents and metadata) of files of interest. 

Much of what is needed to do integrity management has already landed in the mainline, but there are a few remaining pieces. The integrity measurement architecture (IMA) appraisal extension [patch set](<https://lwn.net/Articles/487700/>) from Mimi Zohar and Dmitry Kasatkin fills in one missing piece: storing and validating the integrity measurement of files. A hash of a file's contents and metadata will be stored in the `security.ima` extended attribute (xattr) of the file, and the patch set will create and maintain those xattrs. In addition, it can enforce that the file contents are "correct" when the file is opened for reading or executing based on the integrity values that were stored. 

The integrity subsystem has taken a rather twisted path into the kernel. It was [proposed](<https://lwn.net/Articles/137306/>) as far back as 2005, but the subsystem has been broken up into smaller pieces several times along the way. Much of IMA was added to the kernel in 2.6.30, but another piece, the [extended verification module](<https://lwn.net/Articles/394170/>) (EVM) was not merged until 3.2. Digital signature support was added to EVM in 3.3, and IMA appraisal is currently under review. 

As described on the [Linux IMA web page](<http://sourceforge.net/apps/mediawiki/linux-ima/index.php?title=Main_Page>), the integrity subsystem is meant to thwart various kinds of attacks against the contents of files, both on- and off-line. Unexpected changes to files, particularly executables, may be a sign that the system has been compromised. In addition, the subsystem allows the use of the "Trusted Platform Module" (TPM) to collect integrity measurements and sign them in such a way that the system can "attest" to its integrity. That attestation could be sent to another system to "prove" that the system is intact--only approved code is running. 

Current kernels can generate an integrity measurement of files that are executed, collect and digitally sign them with keys from the TPM (or the kernel keyring), and use that information for remote attestation. EVM adds the ability to thwart offline attacks against the file contents or metadata by hashing the values of the security xattrs of the file (e.g. `security.selinux`, `security.ima`), signing that hash, and storing it as `security.evm`. 

But, there is nothing in place that would stop a running system from executing or reading a file that has been changed. If a file with an IMA hash is opened for reading or executing, the appraisal extension will check to see if the contents match the stored hash. If they don't match, the `ima_appraise` kernel command-line parameter determines what happens. If it is set to "enforce", access to the file is denied, while "fix" will update the IMA xattr with the new value. In addition, "off" can be used to turn off any file appraisal. 

In order to recognize that a file has changed while it is open, the appraisal extension requires the filesystem to support `i_version`, which is a counter that gets incremented any time the file's inode gets updated. Filesystems must be mounted with `i_version` option in order for the appraisal extension to work. That allows the extension to notice the change when the file is closed and either update the xattr or flag the file change as a policy violation. 

In order to get the initial `security.ima` xattrs on files that are to be appraised (by default, all files owned by root), one boots the kernel with `ima_appraise_tcb` (which enables appraisal) and `ima_appraise=fix`, and then by opening all files of interest (e.g. via a `find` command as [suggested](<http://sourceforge.net/apps/mediawiki/linux-ima/index.php?title=Main_Page#Labeling_the_filesystem_with_.27security.ima.27_extended_attributes>) on the IMA web page). 

The IMA appraisal extension will complete the off-line attack detection that EVM provides. Because the extension will create and maintain the `security.ima` xattr, EVM will be able to detect changes to the file contents. 

In response to an earlier version of the patch set, James Morris [asked](<https://lwn.net/Articles/489115/>) if there were any distributions that were planning to use IMA and EVM once all the pieces are in place. George Wilson said that IBM plans to use it internally once distributions have incorporated it. In addition, Ryan Ware and Kasatkin said that the Tizen mobile distribution plans to use it for some product profiles. 

But, before any of that can happen, the appraisal extension needs to find a way to change its locking behavior to get past a [NAK by Al Viro](<https://lwn.net/Articles/489117/>). In the current patches, the final `__fput()` is deferred if a file is closed before `munmap()` is called in kernels using IMA appraisal. Viro is concerned that this changes the locking conditions based on whether the kernel is using IMA or not, which may make locking problems harder to spot. He also said that the overhead is too high for a commonly used path, and that not all of the places where `__fput()` is used were covered by the patch. So far, no solution to the problem has been found, though Viro did [suggest](<https://lwn.net/Articles/489124/>) possibly using a different mutex for changing xattrs, but that it would take a fair amount of code review to determine if that could be done. 

Given that the patch set completes a job started by EVM, and will, for the most part, complete the integrity subsystem, it seems likely that a solution will be found. There are a few lingering pieces of IMA appraisal that are still coming, according to the ["An Overview of the Linux Integrity Subsystem [PDF]" white paper](<http://downloads.sf.net/project/linux-ima/linux-ima/Integrity_overview.pdf>). Two specific pieces are mentioned, one to add digital signature capabilities for vendor-signed files, and another that will protect directory contents (e.g. filenames). While the currently proposed patches may still need some work before they can be considered for the mainline, those working on the integrity subsystem are probably finally starting to see the light at the end of a long tunnel.
