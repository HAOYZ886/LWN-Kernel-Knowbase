---
title: SipHash in the kernel
url: https://lwn.net/Articles/711167/
date: "January 10, 2017"
category: Hashing functions
author: "By Jonathan Corbet January 10, 2017"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 10, 2017

A hash function performs a one-way computation on a set of data, producing a set of bytes that, one hopes, is effectively random and which cannot be used to derive the input data. The kernel uses hash functions in numerous places for everything from the generation of security-sensitive sequence numbers to the implementation of hash tables. The security of those functions is increasingly in doubt, and, seemingly, their performance can be improved as well. The process of replacing these hash functions will begin with the 4.11 kernel, which should see the introduction of the [SipHash](<https://131002.net/siphash/>) pseudo-random function. 

SipHash is the creation of Jean-Philippe Aumasson and ([inevitably](<https://lwn.net/Articles/681615/>)) Daniel J. Bernstein; readers interested in the details can find them in [this paper [PDF]](<https://131002.net/siphash/siphash.pdf>). It was designed with a number of objectives in mind, starting with being a cryptographically secure hash function. In practice, what that means is that it is computationally infeasible to derive the input data from its corresponding hash, or to derive the secret data used in the hashing operation even given the ability to see the output for a set of chosen inputs. Another important objective was speed, especially with smaller inputs. Many existing hash functions have a high setup overhead; that cost matters little when large amounts of data are being hashed, but it hurts for the hashing of smaller inputs. As it happens, many of the hashing operations in the kernel are applied to small chunks of data, so lower overhead would be welcome. 

The list of SipHash users is large and growing; many projects have adopted it in an attempt to defend against [hash-collision attacks](<https://lwn.net/Articles/474912/>). These attacks exploit a known hash function to cause a hash table to degrade into a simple linear list, with potentially devastating effects on performance. The Python language [switched to SipHash](<https://lwn.net/Articles/574761/>) in 2013; other users include various BSD distributions, Perl, Ruby, Rust, and more. This move is [not universally acclaimed](<http://perl11.org/blog/seed.html>), but most seem to see it as a step in the right direction. Thus far, however, the kernel has lacked a SipHash implementation. 

What does the kernel use instead? As might be expected with a large body of code like the kernel, different algorithms are employed in different settings. The generation of TCP sequence numbers, for example, is done using the [MD5](<https://en.wikipedia.org/wiki/MD5>) hash function, which has been regarded as insecure for some time. That is potentially problematic, since an attacker who can predict sequence numbers can interfere with or inject data into network connections. The `get_random_int()` and `get_random_long()` functions used extensively throughout the kernel are also based on MD5. The "syncookies" that can be employed to defend against [SYN flood](<https://en.wikipedia.org/wiki/SYN_flood>) attacks are, instead, generated with [SHA-1](<https://en.wikipedia.org/wiki/SHA-0>) which, while more secure than MD5, is showing its age as well. SHA-1 is also used in the core random-number generator, in the BPF subsystem, and elsewhere. 

Use of those algorithms, however, pales next to the usage of a function called `jhash()` (and its variants), a [Jenkins hash](<https://en.wikipedia.org/wiki/Jenkins_hash_function>) implementation. The kernel contains a lot of hash tables, and, as a general rule, `jhash()` is the hash function used to place data into hash buckets. This function has the advantage of being quite fast, but it makes no claims to cryptographic security. Many in-kernel users include some secret data of their own as a defense against collision attacks. But if the results of the hash are visible to a hacker (and simply listing the contents of the table in order may suffice), then deriving that secret data is a relatively easy task. 

Jason Donenfeld set out to replace all of these hashing functions with [an implementation of SipHash](<https://lwn.net/Articles/710962/>) inside the kernel. SipHash uses an explicit secret key for collision defense, so the first order of business for an in-kernel user is the generation of that key: 

```
#include <linux/siphash.h>
    
        siphash_key_t hash_secret;
        get_random_bytes(&hash_secret, sizeof(hash_secret));
```

The use of `get_random_bytes()` is, according to the documentation, the only proper way to initialize this secret. Thereafter, of course, kernel code should take care not to expose the secret outside of the kernel itself, or the protection against hash collisions will be lost. 

The hashing of data is done with: 

```
u64 siphash(const void *data, size_t len, const siphash_key_t *key);
```

The return value will be a 64-bit hash of the input `data`. There are a number of optimized variants for constant-size input data, but most developers need not worry about those since the generic version will pick one of those at compile time if appropriate. 

SipHash is significantly faster than either MD5 or SHA-1, while producing results that are deemed to be more secure. So the replacement of the older algorithms with SipHash should not be a difficult decision to make. The same is not true for `jhash()`, which is much faster than SipHash. In an attempt to convince `jhash()` users to make the switch, Donenfeld added a "HalfSipHash" variant: 

```
u32 hsiphash(const void *data, size_t len, const hsiphash_key_t *key);
```

This version uses a reduced variant of the SipHash algorithm to produce a smaller and less secure result. Users of `jhash()` who do not want to pay the cost of SipHash might just be convinced to use this version instead, an outcome described by Donenfeld as ""a terrifying but potentially useful possibility"". Potential users will note that, while `hsiphash()` is faster than `siphash()`, it still takes about three times as long as `jhash()` to produce a result. The better security that comes from using it should justify the cost in many settings, but it also seems likely that `jhash()` won't be going away anytime soon. 

The patch set has been through a few revisions, with some relatively small changes being made. The biggest complaint about it seems to have [come from networking maintainer David Miller](<https://lwn.net/Articles/711205/>), who was not entirely happy about moving away from hash functions that are implemented by the CPUs themselves: 

This and the next patch are a real shame, performance wise, on cpus that have single-instruction SHA1 and MD5 implementations. Sparc64 has both, and I believe x86_64 can do SHA1 these days. It took so long to get those instructions into real silicon, and then have software implemented to make use of them as well. 

The interesting thing, as a couple of participants pointed out, is that Linux is not actually using these hashing instructions even on the hardware that supports them. Among other things, they require some setup cost that takes away a lot of the performance benefit, especially for small input data arrays. So the existence of hardware-based implementations is, for now, not relevant. 

In any case, Miller [applied the patches](<https://lwn.net/Articles/711206/>) on January 9, so they should make it into the mainline during the 4.11 merge window. The process of converting at least some of those `jhash()` users has not yet begun, though, and can be expected to take some time.
