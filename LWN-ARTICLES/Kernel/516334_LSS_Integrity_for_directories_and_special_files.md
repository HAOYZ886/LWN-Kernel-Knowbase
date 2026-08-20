---
title: "LSS: Integrity for directories and special files"
url: https://lwn.net/Articles/516334/
date: "September 19, 2012"
category: "Security-Integrity verification"
author: "By Jake Edge September 19, 2012 2012 Kernel Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
September 19, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

Over the last few years, the Linux kernel has added features to measure the integrity of files on disk to protect against offline attacks. The integrity measurement architecture (IMA) was added in the 2.6.30 kernel, and other pieces have followed, but the job is not done. Dmitry Kasatkin gave a presentation at the [2012 Linux Security Summit](<http://kernsec.org/wiki/index.php/Linux_Security_Summit_2012>) (LSS) on an extension to the integrity subsystem to handle the contents of directories as well as various special files. 

Integrity protection is needed to prevent attackers from altering the contents of a filesystem without the kernel's awareness, by removing the disk or booting into an alternative operating system. Runtime integrity is already handled by the existing access control mechanisms, Kasatkin said. Those include discretionary access control (DAC) mechanisms like the traditional Unix file permissions or mandatory access control (MAC) schemes such as those provided by SELinux or Smack. But those mechanisms rely on trusting the access control metadata (e.g. permissions bits or security extended attributes), which can be tampered with in an offline attack. 

IMA measures the integrity of files by calculating a cryptographic hash over the file contents, which is stored in the `security.ima` extended attribute (xattr). IMA can also be used in conjunction with a Trusted Platform Module (TPM) to remotely attest to the integrity of the running system. 

The extended verification module (EVM) was added in 3.2 to protect the inode metadata of files against offline attacks. That metadata includes the security xattrs (including those for SELinux and Smack along with `security.ima`), mode (permissions), owner, inode number, etc. Once again, a hash of the values is used, and EVM stores that as the `security.evm` xattr on the file. 

The digital signature extension was added in the 3.3 kernel to allow the IMA and EVM xattrs to be signed. In addition to storing a hash value in the xattrs, a digital signature of the hash value can also be stored and verified. 

The IMA-appraisal feature, which Kasatkin said is being targeted for 3.7, will inhibit access to files whose IMA hash does not match the contents (i.e. the file has been changed offline). There were some [locking problems](<https://lwn.net/Articles/494158/>) that prevented IMA-appraisal from being merged earlier, but those have been resolved. 

But, all of those pieces don't add up to everything needed for real integrity protection, Kasatkin said. While EVM protects the inode metadata and IMA protects the contents of regular files, there is a missing piece: file names. In Linux, the inode does not contain the file name, as it lives in the directory entries, and the association between a file name and an inode is not protected. 

The result is that files can be deleted, renamed, or moved in an offline attack without being detected by the integrity subsystem. In addition, symbolic links and device nodes are currently unprotected, which means that those files can be added, modified, or removed offline without detection. Various attacks are possible via changing directory entries, he said. One could delete a file required for booting, or restore a backup version (and associated security xattrs) of a program with known vulnerabilities. 

Using two virtual machines, Kasatkin simulated an offline attack by creating files in one VM, then mounting the disk in the other VM and changing some of the files. With the existing integrity code (including IMA-appraisal), he was unable to access files with changed contents in the original VM, but had no problems accessing files that had been renamed or moved (nor were deleted files detected). 

That problem leads to the directory and special file integrity protection that he has [proposed](<https://lwn.net/Articles/512364/>). For directories, two new hooks, `ima_dir_check()` and `ima_dir_update()`, would be added. The former would be called during path lookup (from `may_lookup()`) and would deny access if any directory entries in the path had been unexpectedly altered. When directories are updated in the running system, `ima_dir_update()` would be called to update the integrity measurement to reflect those changes. 

The implementation of the verification starts from the root inode during a path lookup. Nothing happens when the filesystem is mounted, the verification is done lazily during file name lookup. Whenever a dentry (directory cache entry) is allocated for a directory, a call is made to `ima_dir_check()` to verify it. This proposed callback does not break [RCU path walk](<https://lwn.net/Articles/419811/>), so it should not cause scalability problems on larger machines. The integrity measurement is calculated with a hash over the list of entries in the directory, using the inode number, name, type, and offset values for each, and storing the result in `security.ima` on the directory (which is then protected with EVM). 

For special files, like symbolic links and device nodes, there is one new hook that has been added: `ima_link_check()`. It is called during path lookup (`follow_link()`) and for the `readlink()` system call. The measurement is a hash of the target path for symbolic links or the major and minor device numbers for device nodes. Once again, those values are stored in `security.ima` and are verified before access. 

The user-space tools used to set the integrity measurements for image creation also need updating to support the new features. The `evmctl` command (part of the ima-evm-utils package) has added the ability to set the reference hashes for directories and special files. 

Kasatkin then demonstrated the integrity protections of the new code. If a file is moved or removed, the directory holding the file can no longer be accessed, so commands like `ls` or `cd` fail with an `EPERM`. He also presented performance numbers that showed relatively modest decreases compared to IMA/EVM without the directory and special file handling code, but more substantial declines when compared to not having IMA/EVM enabled at all. Interestingly, though, both flavors of IMA/EVM performed better on a file copy test than did a disk encrypted using dm-crypt. Disk encryption is another way to thwart offline attacks, of course. 

It would seem that the kernel integrity subsystem is approaching "completion". The final pieces of the puzzle are now available; Kasatkin and others are hopeful they will be acceptable upstream soon, though he did note that the VFS developers had not yet reviewed the most recent patch set. For those that need this kind of protection for Linux, though, the wait may nearly be over.
