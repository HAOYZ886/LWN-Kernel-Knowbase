---
title: Encrypted file backup for ext4
url: https://lwn.net/Articles/668083/
date: "December 16, 2015"
category: "Filesystems-ext4"
author: "By Jake Edge December 16, 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
December 16, 2015

Backing up files from an encrypted filesystem without having access to the key is a more complicated process than one might think. For one thing, the directory containing the files is likely encrypted as well, so some mechanism to extract an encrypted version of the file name would be needed. A recent [patch set](<https://lwn.net/Articles/667767/>) posted to the linux-ext4 mailing list would provide support for extracting files from an [encrypted ext4 filesystems](<https://lwn.net/Articles/639427/>) for backup or other purposes. 

Ted Ts'o posted the patches on December 10\. They add some `ioctl()` commands and a mount option to allow administrators to extract the information needed to store a copy of the file without having a copy of the encryption key. It would not, however, allow the restoration of the file without the key. According to Ts'o, manipulating both the encrypted directory and file without the user's key is tricky: ""establishing a link means that we have to manipulate both the encrypted directory and the encrypted file, and doing this through the VFS interface is non-trivial"" 

There are three chunks of information that need to be stored so that a file can be restored: the encrypted file data, an encrypted version of the file name, and the metadata that contains the encryption context that is stored as an extended attribute (xattr). Before a privileged process (i.e. one with `CAP_SYS_ADMIN`) can read the encrypted file contents, though, the filesystem must be mounted with a newly added option (`ciphertext_access`). 

The patches allow the encrypted file data to be read using the `O_DIRECT` mechanism. But there is a snag. The file size might not be a multiple of the AES block size (which is used for doing the encryption); reading the short chunk at the end of such a file is not supported by `O_DIRECT`, Ts'o said. Adding a way to do that would require changes to the direct I/O code, which may be controversial. So instead, a "shadow" copy of the inode is used with a file size that works and the encrypted file is padded with zeroes to that size when read. That is something of a hack, Ts'o acknowledged. In addition, the original file size will need to be added to the list of data that gets stored as part of the backup (i.e. along with the encrypted file name and the encryption metadata). 

For the encryption metadata, two `ioctl()` commands have been added: `EXT4_IOC_GET_ENCRYPTION_METADATA` and `EXT4_IOC_SET_ENCRYPTION_METADATA`. As the names imply, they get and set the metadata needed to decrypt the file and file name when the user's key is available. Lastly, the `EXT4_IOC_GET_ENCRYPTED_FILENAME` command will retrieve the encrypted file name. 

Andreas Dilger [asked](<https://lwn.net/Articles/668154/>) why `ioctl()` and `O_DIRECT` were used, rather than virtual xattrs and regular reads. If the key is not present, reading the file could simply return the encrypted contents. That, coupled with virtual xattrs for the other needed pieces, would allow existing tools (e.g. `tar --xattrs`) to back up the files, he said. 

But Ts'o [pointed out](<https://lwn.net/Articles/668155/>) a few problems with that approach. The problem with the file size will need to be dealt with by any tool that is used. And he is unconvinced that the effort needed to handle file restoration without the key will be worth it. Beyond that, though, his choice of `O_DIRECT` was deliberate: direct I/O does not use the page cache, so the cache cannot get "poisoned" with pages of encrypted data. 

In our current design, the page cache for an encrypted file always contains the plaintext. This is necessary so that mmap(2) will work correctly. So if user B is logged onto a machine where user A is active, and user A's keys are in the kernel, we don't determine access control [decisions] for user B based on whether or not the user has the appropriate keyring in a keyring accessible to them. We use Unix permissions to provide security isolation between two users which are logged in. 

He continued by noting that user space should drop the plain text pages from the page cache when user A logs out (and A's keys are removed from the kernel), but that he doesn't want the integrity of an encrypted backup to depend on that. Also, it should still be possible to do an encrypted backup while A is logged in, which can be done when using direct I/O to read the encrypted data. In summary, he said that there are enough reasons that regular backup tools can't be used for this purpose, so ""it's simpler all around to use specific ioctls"". 

Jan Kara [expressed concern](<https://lwn.net/Articles/668159/>) about the shadow inode "hack". He thought that it might not be needed, since direct I/O reads will return any partial block at the end of the file. In addition, the locking implications of effectively having two copies of the inode for a single file were worrisome, he said. So far, Ts'o has not replied, but removing the shadow inode, if that turns out to be possible, will only simplify matters. 

Ts'o also [posted](<https://lwn.net/Articles/668157/>) a C program and script that he has been using to test the changes. While a mechanism to backup encrypted files is certainly needed, the interface proposed feels a little clunky overall, though there may be no real way around that. Ts'o also alluded to another use case that he has for the feature, which he was unable to disclose, that may partly be guiding the interface as well. Given that he is the ext4 maintainer, though, it seems likely we will see some kind of change along these lines before long.
