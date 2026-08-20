---
title: The return of EVM
url: https://lwn.net/Articles/394170/
date: "June 30, 2010"
category: "Integrity measurement architecture; Security-Integrity verification"
author: "By Jake Edge June 30, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
June 30, 2010

The integrity measurement architecture (IMA) has been a part of Linux for roughly a year now--it was merged for 2.6.30--and it can be used to attest to the integrity of a running Linux system. But IMA can be subverted by "offline" attacks, where file data or metadata is changed out from under IMA. Mimi Zohar has proposed the [extended verification module](<http://lwn.net/Articles/393673/>) (EVM) patch set as a means to protect against these offline attacks. 

In its default configuration, IMA calculates hash values for executables, files which are `mmap()`ed for execution, and files open for reading by root. That list of hashes is consulted each time those files are accessed anew, so that unexpected changes can be detected. In addition, IMA can be used with the trusted platform module (TPM) hardware, which is present in many systems, to sign a collection of these hash values in such a way that a remote system can verify that only "trusted" code is running (remote attestation). 

But an attacker could modify the contents of the disk by accessing it under another kernel or operating system. That could potentially be detected by the remote attestation, but cannot be detected by the system itself. EVM sets out to change that. 

One of the additions that comes with the EVM patch set is the [integrity appraisal extension](<https://lwn.net/Articles/394362/>), which maintains the file's integrity measurement (hash value) as an extended attribute (xattr) of a file. The `security.ima` xattr is used to store the hash, which gets compared to the calculated value each time the file is opened. 

[EVM itself](<https://lwn.net/Articles/394364/>) just calculates a hash over the extended attributes in the `security` namespace (e.g. `security.ima`, `security.selinux`, and `security.SMACK64`), uses the TPM to sign it, and stores it as the `security.evm` attribute on the file. Currently, the key to be used with the TPM signature gets loaded onto the root keyring by `readevmkey`, which just prompts for a password at the console. Because an attacker doesn't have the key, an offline attack cannot correctly modify the EVM xattr when it changes file data. Securing the key is important, so future work will entail using TPM sealed keys and encrypted symmetric keys so that the plaintext EVM key will never be visible to user space. 

With all of that in place, a system administrator can be sure that the code running on the system is the same as that which was measured. Presumably, the initial measurement is done from a known good state. After that, any offline attack would need to either modify a file's contents, which would cause the IMA comparison to fail, or modify its security xattrs, which would cause the EVM comparison to fail. 

These patches have been bouncing around in various forms for five years or more; we first [looked](<http://lwn.net/Articles/160157/>) at EVM in 2005. The EVM patch describes some of the changes that EVM has undergone along the way: ""EVM has gone through a number of iterations, initially as an LSM module, subsequently as a LIM [[Linux integrity module](<http://lwn.net/Articles/287790/>)] integrity provider, and now, when co-located with a security_ hook, embedded directly in the security_ hook, similar to IMA."" That evolution reflects both changes suggested in the review process as well as a realization that, since Linux security modules (LSMs) don't [stack](<http://lwn.net/Articles/393008/>), it would be impossible to have both EVM and SELinux, say, in one kernel. That led to adding IMA, and now EVM, as calls out from the appropriate security hooks or VFS code. 

For EVM, the hooks affected are `security_inode_setxattr()`, `security_inode_post_setxattr()`, and `security_inode_removexattr()`, each of which embeds a call to the appropriate `evm_*` function. The `evm_inode_setxattr()` function protects the `security.evm` xattr from modification unless the `CAP_MAC_ADMIN` capability is held. The other two calls update the EVM hash associated with a file when xattrs are changed. 

The patches aren't too intrusive outside of the security subsystem, though they do touch some other areas. Two new generic VFS calls (`vfs_getxattr_alloc()` and `vfs_xattr_cmp()`) were added to simplify xattr handling. Because various additional file attributes (beyond just the security xattrs, like inode number, uid, mode, and so on) are used in the EVM hash, changes to those need to cause a recalculation, which necessitated changes `fs/attr.c`. And so on. 

There are few comments on this iteration of the EVM patches. The idea has been through several rounds of review over the years and the patches have picked up an ACK from Serge E. Hallyn. EVM closes the offline attack hole in the protection that IMA provides and would thus seem to make a good addition the mainline kernel. For those who want to try it out now, there are [instructions](<http://linux-ima.sourceforge.net/#EVM>) available on the [Linux integrity subsystem](<http://linux-ima.sourceforge.net/>) web page. Unless major complaints appear, one would think that EVM might well be a candidate for 2.6.36.
