---
title: "Adiantum: encryption for the low end"
url: https://lwn.net/Articles/776721/
date: "January 16, 2019"
category: Cryptography
author: "By Jake Edge January 16, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
January 16, 2019

Low-end devices bound for developing countries, such as those running the [Android Go edition](<https://developer.android.com/google-play/guides/android-go-edition/>), lack encryption support because the hardware doesn't provide any cryptographic acceleration. That means users in developing countries have no protection for the data on their phones. Google would like to change that situation. The company worked on adding the [Speck cipher](<https://en.wikipedia.org/wiki/Speck_\(cipher\)>) to the kernel, but [decided against using it](<https://lwn.net/Articles/761992/>) because of opposition due to Speck's origins at the US National Security Agency (NSA). As a replacement, the Adiantum encryption mode was developed; it has been merged for Linux 5.0. 

Eric Biggers has been spearheading the effort; he [posted](<https://lwn.net/ml/linux-kernel/20181117012631.23528-1-ebiggers@kernel.org/>) version 4 of the Adiantum patch set in mid-November and it was pulled by kernel crypto maintainer Herbert Xu shortly thereafter; it will appear in the 5.0 kernel. Meanwhile Speck was removed from the kernel in 4.20 for lack of any maintainer or users. The Adiantum patch description is lengthy and informative, but there is also a [paper](<https://eprint.iacr.org/2018/720>) by Biggers and Paul Crowley (who did much of the work in coming up with Adiantum and its predecessor HPolyC). Incidentally, the paper notes that the name "Adiantum" is the genus of the [maidenhair fern](<https://en.wikipedia.org/wiki/Adiantum>). 

Adiantum is intended to be a choice for the encryption and decryption algorithm for disk encryption on Linux systems. It can be used either for block-level encryption as part of [dm-crypt](<https://en.wikipedia.org/wiki/Dm-crypt>) or for file and directory encryption as part of [fscrypt](<https://www.kernel.org/doc/html/v4.18/filesystems/fscrypt.html>). Adiantum and its supporting crypto primitives needed to be added to the kernel so that it can be used from these kernel subsystems. Most of the 14-part patch set is adding various crypto primitives used by Adiantum. 

It's worth noting that Adiantum is not a new encryption algorithm as such; instead, it is a repackaging of the ChaCha stream cipher that makes it useful for disk encryption. That makes reasoning about its security relatively straightforward: 

Adiantum is a construction, not a primitive. Its security is reducible to that of XChaCha12 and AES-256, subject to a security bound; the proof is in Section 5 of our paper. Therefore, one need not "trust" Adiantum; they only need trust XChaCha12 and AES-256. 

In this way, the authors have tried, with apparent success, to avoid the trust issues that surrounded Speck. 

Many low-end, inexpensive devices (e.g. mobile phones for the developing world) and even some smartwatches are shipped with older or less powerful Arm CPUs that lack the cryptographic extensions that more recent processors have. The goal was to find a way to encrypt filesystem data on those devices and, crucially, to be able to decrypt it quickly enough that users will not be annoyed by the performance—or have their batteries unduly impacted. Speck mostly fit the bill, but it turns out that Adiantum is even faster (roughly 30%), so the political issues that made Speck untenable turned out to be a boon for users. 

HPolyC was the original algorithm that Biggers and Crowley were planning to use as a Speck replacement; it was already faster than Speck but some further refinements led to Adiantum and even better performance. The main change between HPolyC and Adiantum is the hash function used. The [Poly1305](<https://en.wikipedia.org/wiki/Poly1305>) message authentication code (MAC) hash family is used by both, but Adiantum first uses a hash from the [NH family of hashes](<https://en.wikipedia.org/wiki/UMAC#NH_hash-function_family>) to effectively compress the data by 32x first. After that, Adiantum uses Poly1305. 

Both Poly1305 and NH are families of hash functions that are deemed "almost universal". A [universal hash](<https://en.wikipedia.org/wiki/Universal_hashing>) family has the property that it minimizes collisions even if the input is controlled by an adversary. Each member of the family is generally able to spread the input over a wide number of buckets but any single member will be susceptible to a [preimage attack](<https://en.wikipedia.org/wiki/Preimage_attack>). By choosing one of the family members at random, that kind of attack is thwarted. 

Using NH in addition to Poly1305 does reduce the key agility of Adiantum; the paper recommends using HPolyC in applications that need to be able to switch keys quickly. For performance reasons, NH is easily implemented in [SIMD](<https://en.wikipedia.org/wiki/SIMD>) assembly (such as [Arm NEON](<https://en.wikipedia.org/wiki/ARM_architecture#Advanced_SIMD_\(NEON\)>)) but the more complicated Poly1305 is written in C, which aids portability. 

The encryption cipher used is XChaCha12, which is a block cipher based on the ChaCha family of stream ciphers. It uses 12 rounds, as the name would imply, which is lower than the 20-round ChaCha that is commonly used. The best-known attacks against ChaCha are for the seven-round variant, so ChaCha12 still provides a strong cipher. Two rounds of XChaCha12 are followed by an AES-256 encryption, but of just 16 bytes. AES is often used for disk encryption on higher-end devices because their processors provide AES acceleration, but it is far too slow and power hungry to run on low-end devices. 

[According to Biggers](<https://lwn.net/ml/linux-kernel/20181117012631.23528-15-ebiggers@kernel.org/>), this provides a better security margin than HPolyC or AES. In addition, Adiantum has the property that changing a single bit in the input completely scrambles the block, unlike other modes (e.g. [XTS](<https://en.wikipedia.org/wiki/Disk_encryption_theory#XTS>)), where it will only affect 16 bytes in the block. 

Adiantum is a length-preserving encryption, which is important for disk encryption. It would be ideal to store random nonces along with each block of ciphertext, Biggers said, but that requires another layer (such as [dm-integrity](<https://www.kernel.org/doc/Documentation/device-mapper/dm-integrity.txt>)) to manage the extra data per block. That negatively impacts performance, so, at least for now, length-preserving encryption is needed. 

"Encryption for all" is an explicit goal in various domains; it has driven the push for "HTTPS Everywhere", for example. It is nice to see work being done to ensure that people in developing countries will be able to secure their data on what may well be their only computing device: their mobile phone. One hopes that Adiantum and HPolyC will be adopted widely—in Android and beyond.
