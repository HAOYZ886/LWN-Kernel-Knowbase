---
title: Authenticated Btrfs
url: https://lwn.net/Articles/818842/
date: "April 30, 2020"
category: "Filesystems-Btrfs; Security-Integrity verification"
author: "By Jonathan Corbet April 30, 2020"
---

By **Jonathan Corbet**  
April 30, 2020

Developers who are concerned about system integrity often put a fair amount of effort into ensuring that data stored on disk cannot be tampered with without being detected. Technologies like [dm-verity](<https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/verity.html>) and [fs-verity](<https://lwn.net/Articles/790185/>) are attempts to solve this problem, as is the recently covered [integrity policy enforcement security module](<https://lwn.net/Articles/817472/>). More Recently, Johannes Thumshirn has posted [a patch series](<https://lwn.net/ml/linux-fsdevel/20200428105859.4719-1-jth@kernel.org/>) adding filesystem-level authentication to Btrfs; it promises to provide integrity with a surprisingly small amount of code. 

Integrity-verification code at the filesystem or storage level generally works by calculating (and storing) checksums of each block of data. When it comes time to read that data, the checksum is calculated anew and compared to the stored value; if the two match, one can be confident that the data has not been modified (or corrupted by the hardware) since the checksum was calculated. If there is reason to believe that the stored checksum is what the creator of the data intended, then the data, too, should be as intended. 

Solutions like dm-verity and fs-verity work by storing checksums apart from the data; fs-verity, for example, places the checksum data in a hidden area past the end of the file. The developers of more modern filesystems, though, have generally taken the idea that storage devices are untrustworthy (if not downright malicious) to heart; as a result, they design the ability to calculate, store, and compare checksums into the filesystem from the beginning. Btrfs is one such filesystem; as can be seen from [the on-disk format documentation](<https://btrfs.wiki.kernel.org/index.php/On-disk_Format>), most structures on disk have a checksum built into them. Checksums for file data is stored in a separate tree. So much of the needed infrastructure is already there. 

Checksums in Btrfs, though, were primarily intended to catch corruption caused by storage hardware. The thing about hardware is that, while it can be creative indeed in finding new ways to mangle data, it's generally not clever enough to adjust checksums to match. Attackers tend to be a bit more thorough. So the fact that a block of data stored in a Btrfs filesystem matches the stored checksum does not, by itself, give much assurance that the data has not been messed with in a deliberate way. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

To gain that assurance, Btrfs needs to use a checksum that cannot readily be altered by an attacker. Btrfs already supports a number of checksum algorithms, but none of them have that property. So the key to adding the needed sort of authentication to Btrfs is to add another checksum algorithm with the needed assurance. Thumshirn chose to add an [HMAC](<https://en.wikipedia.org/wiki/HMAC>) checksum based on SHA-256. 

Calculating an HMAC checksum for a block of data requires a secret key; without the key, the code can neither calculate checksums nor verify those that exist. This key must be provided when the filesystem is created; an example from the patch set reads like this: 

```
mkfs.btrfs --csum hmac-sha256 --auth-key 0123456 /dev/_disk_
```

Here, `0123456` is the authentication key to be used with this filesystem. 

This key must also be provided when the filesystem is mounted; that does not happen directly on the command line, though. Instead, the key must be stored in the kernel's trusted keyring; the name of that key is then provided as a mount option. This (hopefully) keeps the key itself from appearing in scripts or configuration files; instead, the key must come from a trusted source at system boot time. 

That is really about all there is to it. An attacker who lacks the trusted key can still read the filesystem, but they cannot make changes without breaking the checksums — and that will cause any such changes to be detected the next time the affected data is read. It is worth noting, though, that an attacker who can compromise the kernel can access the key or just have the kernel write the desired changes directly to the filesystem. Solutions like fs-verity, instead, usually don't allow the key anywhere near production systems; that makes the protected files read-only, but that is usually the intent anyway. So authenticated Btrfs is suitable for deterring offline attacks, but it may not be able to protect against as wide a range of attacks as some other technologies. 

On the other hand, authenticated Btrfs requires minimal changes to the Btrfs code, and doesn't require the interposition of any other layers between the file system and the storage device. It may well prove useful for a range of use cases. The patch set is relatively young, though, and has not yet received much in the way of review comments. The real test will happen once developers find the time to give these changes a close look.
