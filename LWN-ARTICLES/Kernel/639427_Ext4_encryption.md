---
title: Ext4 encryption
url: https://lwn.net/Articles/639427/
date: "April 8, 2015"
category: "Filesystems-ext4; Security-Filesystem encryption"
author: "By Jonathan Corbet April 8, 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 8, 2015

For reasons that should be reasonably obvious, there is an increasing level of awareness of the wisdom of encrypting sensitive data stored on devices — especially on devices that, like a phone handset, are easily stolen or lost. In current kernels, encrypting a filesystem requires the use of an add-on module like [eCryptfs](<https://lwn.net/Articles/156921/>) or [dm-crypt](<http://en.wikipedia.org/wiki/Dm-crypt>). These modules work, but they can have an adverse effect on filesystem performance as a result of the way they are implemented. Performance is important; problems in this area are widely cited as the reason for Google's decision to back off from its plan to encrypt filesystems by default in the Android "Lollipop" release. Linux might be able to provide a filesystem with better performance if encryption were built into the filesystem itself, but, currently, not even Btrfs has encryption as an option. 

Change is afoot, however; it takes the form of [a set of patches adding encryption to the ext4 filesystem](<https://lwn.net/Articles/639262/>). They were posted by ext4 maintainer Ted Ts'o, but the lead developer behind the work is Michael Halcrow — the same developer who added eCryptfs to the kernel ten years ago. The ext4 work, though, reflects some of the lessons that have been learned in the meantime. 

Performance suffers in eCryptfs as a result of the stacked nature of the filesystem. Imagine a system running eCryptfs over ext4 now; if a process wants to read a page from an encrypted file, eCryptfs must first instruct ext4 to read that page into the page cache. It then decrypts the data — into another page-cache page. The extra copies of the data can consume a lot of memory and slow things down unnecessarily. Putting encryption support directly into ext4 can eliminate much of that waste. 

Encryption in ext4 is a per-directory-tree affair. One starts by setting an encryption policy (using an `ioctl()` call) for a given directory, which must be empty at the time; that policy includes a master key used for all files and directories stored below the target directory. Each individual file is encrypted with its own key, which is derived from the master key and a per-file random nonce value (which is stored in an extended attribute attached to the file's inode). File names and symbolic links are also encrypted. 

The keys used by processes to access the encrypted directory tree are stored in the kernel's keyring as "logon" keys, meaning that user space can create them, but it is not allowed to read the value of the keys. (The kernel's key-management functionality is beyond the scope of this article; see [`Documentation/security/keys.txt`](<https://lwn.net/Articles/639523/>) for an overview of how it works). If a user-space process has the requisite master key in its per-process keyring, it can access an encrypted directory as usual. In the absence of the key, though, things are different. Directories can still be read (if the normal permissions and security module policy allow, of course), but the file names will all be encrypted, so the result may not be particularly satisfying. It will be possible to determine how many files are in the directory, their sizes, and their permissions, but not their names or contents. Attempts to open a file (for read or write) without access to the key will simply fail. It is still possible to delete encrypted files, though, if the permissions allow. 

If a process with access to the appropriate key reads a page from a file, the filesystem code starts by allocating a separate bounce buffer. The encrypted data is read into the buffer, then decrypted into the page cache. Writes work similarly: the page being written is read from persistent storage and decrypted if necessary; then the new data is written, the data is encrypted into a bounce buffer, and written to permanent storage. Some extra memory is used during the actual encryption and decryption operations, but then it is immediately returned to the system, so overall memory use is significantly reduced relative to eCryptfs or dm-crypt. 

Keeping the plain text of an encrypted file in memory has some obvious risks associated with it; if an attacker can get at that memory, all of the work put into encrypting the file on disk is for nothing. To an extent that risk just has to be accepted; the developers are not attempting to make a system that is resistant to attacks when it is hibernated, for example. Still, efforts have been made to clear plain-text data out of memory when it is no longer needed in an attempt to mitigate that risk somewhat. The developers note, though, that if an attacker can make changes to an encrypted filesystem that is subsequently mounted by the user, all bets are off. So ext4 encryption can protect a lost or stolen device, but protecting a device that has been covertly modified is beyond its threat model. 

The code currently uses AES-256-XTS as the encryption algorithm for file contents, while AES-256-CBC+CTS is used for file names. The code is designed with the idea that, at some point, it will be desirable to change to a different encryption scheme; care has been taken to avoid wiring the specific algorithm too deeply into the filesystem. 

While Google prefers ext4 as the filesystem to use on Android systems, not all Android devices use it. So it is worth noting that Ted has been talking with the maintainer of the F2FS flash-oriented filesystem to get the same `ioctl()` interfaces implemented there. That would allow Android systems to use encrypted storage on either filesystem without the need for any filesystem-specific code. 

This code is marked as experimental in the current patch set, but it may not stay that way for long. There is already user-space code to make use of this feature in the Android open-source repository, and, according to Ted, it will be included in the next major Android release. As of this writing, it has also found its way into linux-next, suggesting that it is intended for the 4.1 merge window. Some developers [think that may be premature](<https://lwn.net/Articles/639433/>), though, since the code has just now surfaced. Filesystem changes in general merit a high level of review, given the severe consequences of getting something wrong at that level of the system. Security-relevant code needs even more review, of course. Until that review has happened, developers may well feel nervous about shipping these particular changes in a mainline kernel release. 

This obstacle will likely be overcome before too long; at that point Linux will have native encryption support in a major filesystem for the first time. In a period where many people are concerned about the security of their data, that can only be a good thing. 

(See [this document](<https://docs.google.com/document/d/1ft26lUQyuSpiu6VleP70_npaWdRfXFoNnB8JYnykNTg>) for some more information on the design of the ext4 encryption mechanism).
