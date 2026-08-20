---
title: "Yet another try for fs-verity"
url: https://lwn.net/Articles/790185/
date: "June 3, 2019"
category: "Filesystems-fs-verity; Security-Integrity verification"
author: "By Jonathan Corbet June 3, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 3, 2019

The fs‑verity mechanism has its origins in the Android project; its purpose is to make individual files read-only and enable the kernel to detect any modifications that might have been made, even if those changes happen offline. Previous fs‑verity implementations have run into criticism in the development community, and none have been merged. A [new version of the patch set](<https://lwn.net/ml/linux-fsdevel/20190523161811.6259-1-ebiggers@kernel.org/>) was posted on May 23; it features a changed user-space API and may have a better chance of getting into the mainline. 

Fs‑verity works by associating a set of hashes with a file; the hash values can be used to check that the contents of the file have not been changed. In current implementations, the hashes are stored in a [Merkle tree](<https://en.wikipedia.org/wiki/Merkle_tree>), which allows for quick verification when the file is accessed. The tree itself is hashed and signed, so modifications to the hash values can also be detected (and access to the file blocked). The intended use case is to protect critical Android packages even when an attacker is able to make changes to the local storage device. 

Previous versions of the fs‑verity patches [ran aground](<https://lwn.net/Articles/775872/>) over objections to how the API worked. To protect a file, user space would need to generate and sign a Merkle tree, then append that tree to the file itself, aligned to the beginning of a filesystem block. After an `ioctl()` call, the kernel would hide the tree, making the file appear to be shorter than it really was, while using the tree to verify the file's contents. This mechanism was seen as being incompatible with how some filesystems manage space at the end of files; developers also complained that it exposed too much about how fs‑verity was implemented internally. In the end, an [attempt to merge this code for 5.0](<https://patchwork.kernel.org/patch/10745561/>) was not acted upon, and fs‑verity remained outside of the mainline. 

The new patch set addresses these concerns by moving the generation of the Merkle tree into the kernel and hiding the details of where this tree is stored. To enable fs‑verity protection for a file, a user-space application starts by opening the file in question. Despite the fact that this operation changes the file (by adding the protection and making the file read-only), this file descriptor must be opened for read access only. Then, the new `FS_IOC_ENABLE_VERITY` `ioctl()` command is invoked on this file; the application passes in a structure that looks like this: 

```
struct fsverity_enable_arg {
    	__u32 version;
            __u32 hash_algorithm;
            __u32 block_size;
            __u32 salt_size;
            __u64 salt_ptr;
            __u32 sig_size;
            __u32 __reserved1;
            __u64 sig_ptr;
            __u64 __reserved2[11];
        };
```

The `version` field must be set to one; it is there to allow different fs‑verity implementations in the future. Similarly, the reserved fields must all be set to zero. `hash_algorithm` tells the kernel which algorithm to use for hashing the file's blocks; the only supported values at the moment are `FS_VERITY_HASH_ALG_SHA256` and `FS_VERITY_HASH_ALG_SHA512`. The block size for the hash is set in `block_size`; it must match the filesystem block size. If `salt_size` and `salt_ptr` are set, they provide a "salt" value that is prepended to each block prior to hashing. A digital signature for the hash of the file can optionally be added using `sig_ptr` and `sig_size`; more on that shortly. 

This `ioctl()` call will read through the entire file, generating the Merkle tree and storing it wherever the filesystem thinks is best. If the file is large, this operation can take some time; it can be interrupted with a fatal signal, leaving the file unchanged. Enabling fs‑verity will fail if there are any open, write-enabled file descriptors for the target file. 

After the operation succeeds, the file will be in the fs‑verity mode. Opens for write access will fail, even if the file's permission bits would otherwise allow writing. Some metadata can still be changed, though, and the file can be renamed or deleted. Any attempt to read from the file will fail (with `EIO`) if the data of interest does not match the stored hash. If user space is counting on fs‑verity protection, though, it should, after opening the file, verify that this protection is present with the `FS_IOC_MEASURE_VERITY` `ioctl()` call, which takes a pointer to this structure: 

```
struct fsverity_digest {
    	__u16 digest_algorithm;
    	__u16 digest_size; /* input/output */
    	__u8 digest[];
        };
```

If the file is protected with fs‑verity, this structure will be filled in with summary hash information. 

User space can use that information to verify that the `digest` data matches expectations; without that test, an attacker could substitute a new file with hostile contents and a matching Merkle tree. Alternatively, this digest can be signed and the kernel will verify that it matches at access time. What must actually be signed is this structure: 

```
struct fsverity_signed_digest {
    	char magic[8];                  /* must be "FSVerity" */
    	__le16 digest_algorithm;
    	__le16 digest_size;
    	__u8 digest[];
        };
```

The digest information can be obtained from the kernel using the `FS_IOC_MEASURE_VERITY` `ioctl()` described just above. So one way to add a signature to an fs‑verity file would be to create the file once, enable fs‑verity on the file _without_ a signature, obtain the digest information, then create and enable the file a second time with the signature data. In practice, files to be protected this way (such as Android package files) will probably be shipped with the associated signature data, so this two-step process will not be necessary on the target systems. 

The final piece for signature verification is the provision of a public key to verify against. The fs‑verity subsystem creates a new keyring (called `.fs‑verity`); a suitably privileged user can add certificates to this keyring for use in file verification. The signing key, of course, should not be on the target system at all; assuming that the attacker cannot obtain that key by other means, verification against the public key should provide assurance that the file has not been modified. 

The ext4 and F2FS filesystems are supported in the current patch set. See [the extensive documentation file](<https://lwn.net/ml/linux-fsdevel/20190523161811.6259-2-ebiggers@kernel.org/>) provided for the patch set for a lot more details on how it all works. Some kernel features are added without sufficient documentation; fs‑verity does not look like it will be one of those. 

Previous versions of the patch set have generated a lot of (sometimes heated) discussion. This time, the response has been silent, prompting Eric Biggers (the author of this work) to [ask](<https://lwn.net/ml/linux-fsdevel/20190530185414.GD70051@gmail.com/>) if anybody has any comments. Unless somebody shows up with objections, the logical conclusion is that the biggest concerns have been addressed and that fs‑verity may be on track for merging into the 5.3 kernel.
