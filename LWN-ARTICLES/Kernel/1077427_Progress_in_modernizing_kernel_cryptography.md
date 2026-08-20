---
title: Progress in modernizing kernel cryptography
url: https://lwn.net/Articles/1077427/
date: "July 8, 2026"
category: Cryptography
author: "By Joe Brockmeier July 8, 2026 LSS NA"
---

By **Joe Brockmeier**  
July 8, 2026

* * *

[LSS NA](<https://lwn.net/Archives/ConferenceByYear/#2026-Linux_Security_Summit_North_America>)

At the 2026 [Linux Security Summit North America](<https://events.linuxfoundation.org/linux-security-summit-north-america/>), Eric Biggers spoke about some of the problems with the kernel's cryptography framework, as well as the recent progress in adding library APIs to allow developers to use cryptographic functions without using the traditional crypto API. He walked through a couple of examples to demonstrate the frailty of the original API and showed how the new library API made life easier for developers and kernel maintainers.

Biggers began by introducing himself. He is a maintainer of the [crypto](<https://docs.kernel.org/crypto/libcrypto.html>) and [cyclic redundancy check](<https://docs.kernel.org/core-api/kernel-api.html#crc-functions>) (CRC) library code in the Linux kernel, as well as of the [fscrypt](<https://www.kernel.org/doc/html/latest/filesystems/fscrypt.html>) library and [fs-verity](<https://docs.kernel.org/filesystems/fsverity.html>) support layer. He said that he was grouping CRCs in with crypto because ""they are very similar from an implementation perspective"" and the code was for the kernel itself to use--user space has its own code. ""It's for all the kernel features that use these algorithms and need to execute them in kernel mode."" The use cases, he said, range from storage or network encryption to generating random numbers, checking firmware integrity, protecting against denial-of-service attacks, and more.

#### Traditional API

The Linux kernel has had what Biggers called the traditional crypto API, found in the top-level [`crypto`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/crypto>) directory in the kernel tree, [since 2002](<https://lwn.net/Articles/14006/>). It includes cryptographic algorithms as well as some non-cryptographic algorithms such as CRC. Unfortunately, he said, ""this traditional crypto API is not working well. It's complex, it's hard to use, and it's often quite slow"". That has always been true, he said, but it has gotten worse over time.

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The traditional API does not match what most kernel developers want, he said, and has not kept up with hardware: ""Specifically, it isn't well optimized for the CPU-based acceleration that modern systems use."" He also said that the framework was outdated and that the API does not work well for newer algorithms and implementation strategies.

To prove the point, he said he would show a couple of examples of using a crypto algorithm in the kernel and ""why the traditional crypto API isn't all that great"". He began with computing a message authentication code (MAC) with a key and some data using HMAC-SHA256, ""one of the most common MAC algorithms"". The code is from on slides five and six in his [presentation](<https://hosted-files.sched.co/lssna2026/5e/Linux%20Security%20Summit_%20Modernizing%20Kernel%20Cryptography.pdf>).

```
static int calc_hmac(const u8 *key, size_t keylen, const u8 *data, size_t datalen, u8 out[32])
        {
        	struct crypto_shash *tfm;
        	int err;
        
        	tfm = crypto_alloc_shash("hmac(sha256)", 0, 0);
        	if (IS_ERR(tfm)) {
        	   pr_err("Failed to allocate hmac(sha256): %ld\n", PTR_ERR(tfm));
        	   return PTR_ERR(tfm);
        	}
```

To use the traditional crypto API to get the MAC value, he explained, the first thing to do is to dynamically allocate a `crypto_shash` object for HMAC-SHA256 by passing the name of the algorithm as a string. Then, one has to check for errors, ""since this step can and often does fail"".

```
err = crypto_shash_setkey(tfm, key, keylen);
    	if (!err) {
    	        SHASH_DESC_ON_STACK(desc, tfm);
    
    		desc->tfm = tfm;
    		err = crypto_shash_digest(desc, data, datalen, out);
    		shash_desc_zero(desc);
    	}
    	if (err)
    		pr_err("Failed to calculate HMAC-SHA256 value: %d\n", err);
    	crypto_free_shash(tfm);
    	return err;
```

[ ![\[Eric Biggers speaking\]](https://static.lwn.net/images/2026/Eric_Biggers-sm.png) ](<https://lwn.net/Articles/1081206/#eric>)

The next step is to set the key on the `crypto_shash` object. That step can also fail, so more error-checking code is required. Then it is necessary to allocate a synchronous hash descriptor; he said that the easiest way to do that is to use the `SHASH_DESC_ON_STACK()` macro. After that, initialize the hash descriptor by setting the pointer to the `crypto_shash`. After all that, then it's time to call the function that actually computes the MAC value. Once that is done, clear sensitive data from the stack by [zeroizing](<https://en.wikipedia.org/wiki/Zeroisation>) the hash descriptor, check for errors, and free the `crypto_shash`.

Biggers said that this code ""sort of works"" but it is common for bugs to be found after such code is merged because allocating the `crypto_shash` fails on some systems. The fix for that is to make sure that `CRYPTO_HMAC` and `CRYPTO_SHA256` are enabled in the Kconfig options. ""This part is often overlooked because the algorithms are loaded by name, so there's no link-time dependency on them."" The code builds fine, but fails at run time and only on some systems. And if a developer wants the code to run in an early init call? ""They're still out of luck; there's no way to do that with the traditional API because the crypto API has to initialize itself first.""

Performance suffers as well, he said. He had benchmarked the operation on an x86_64 system running the 6.12 kernel, which predates some recent optimizations. ""It turns out only 38% of the time was spent actually doing the core SHA-256 computation. Most of the time was actually spent on the overhead of the traditional crypto API"".

The better way, he said, was to just add a library function that directly implements HMAC-SHA256 without going through the traditional API. ""It's much easier to use and much more efficient too."" The example (from slide 11) is shown below:

```
hmac_sha256_usingrawkey(key, key_len, data, data_len, out);
```

He said that he had [added this function in the 6.17 kernel](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=077833cd6009>), and that the new APIs were already being used in quite a few places in the kernel. The functions always succeed, he said, and callers don't need to handle errors. ""They also work in all contexts in the kernel, and they just use standard link-time dependencies so that they don't fail at run time."" In addition, it's also faster, particularly with shorter inputs. About 2.5 times as fast as the same operation he had mentioned previously. ""I also added a FIPS-140 cryptographic algorithm self-test, so it should work fine with FIPS certifications"".

This is not a new approach, he said; for example, crypto libraries were added in 2019 for the algorithms used by [WireGuard](<https://en.wikipedia.org/wiki/WireGuard>). Libraries have also been the norm outside the crypto subsystem; he used [`memcpy()`](<https://man7.org/linux/man-pages/man3/memcpy.3.html>) as an example. ""It's just a function that does the thing you want, you don't have to dynamically allocate a memory copier object by name and then call several additional functions"".

He briefly showed another example of encrypting data using [ChaCha20-Poly1305](<https://en.wikipedia.org/wiki/ChaCha20-Poly1305>), an authenticated encryption with associated data (AEAD) algorithm. It takes a key, a nonce, plain text, and associated data as input, and produces a ciphertext.

Using the traditional crypto API, the caller has to start by allocating an AEAD transformation object and setting a key: ""as usual, both can fail, so you have to write error-handling code, and I hope you remember to test your error-handling code, too. I'm sure everyone does that"". He noted that it was also necessary to put data in scatter-gather lists instead of standard buffers; he displayed the example code (slide 15) of setting up the scatter list:

```
struct scatterlist src_sg[2], dst_sg[2];
        sg_init_table(src_sg, ARRAY_SIZE(src_sg));
        sg_init_table(dst_sg, ARRAY_SIZE(dst_sg));
        // The below assumes that none of ad, src, and dst points to vmalloc memory
        sg_set_buf(&src_sg[0], ad, ad_len);
        sg_set_buf(&src_sg[1], src, data_len);
        sg_set_buf(&dst_sg[0], ad, ad_len);
        sg_set_buf(&dst_sg[1], dst, data_len + crypto_aead_authsize(tfm));
```

That was the simplest case where the input and output addresses were already in the kernel's direct mapping. ""However, if any of your data is in the stack or elsewhere in the `vmalloc` region, good luck: because in those cases you'd actually need much more complex code to set up the scatter lists correctly."" Biggers said that he had yet to see a single case where someone actually submitted the correct code on the first try without needing fixes.

He added that the request may complete asynchronously, so callers needed to ensure they waited to free their data after submitting the request to complete, lest they create a use-after-free vulnerability.

The library example, once again, was a much simpler endeavor:

```
chacha20poly1305_encrypt(dst, src, src_len, ad, ad_len, nonce, key);
```

All that is required is to call a function that directly implements the algorithm the caller wanted. ""As before, it's synchronous, returns `void`, and it works in any context."" He noted that this function was added in 2019 to support WireGuard, so it was not new. His recent work has been focused on hashing and MACs instead, but this function provided a great example because it demonstrated that the traditional API suffered from a lot of complexity that was no longer needed.

#### Where we are today

As of the 7.1 kernel, the crypto and CRC libraries support a lot of algorithms, he said, and displayed a impressive list (slide 19) that are now supported. Roughly half were added in the last year and a half. But there are still some important ones that are missing, such as the AES encryption modes. ""But I'm working on that.""

The libraries provide a separate set of functions for each algorithm, which allows them to provide a good API for each one: ""just whatever is simplest, easiest to use, and the most efficient"". That means that the library works best for in-kernel users who need a single algorithm, he said, which is the most common case. If a kernel feature needs support for multiple algorithms, ""you just dispatch to the different functions"". That might sound inconvenient, but it is still significantly better than the traditional API. But that API remains available, and has been reimplemented using the libraries where possible.

In the last year and a half, Biggers said, he (along with co-maintainers Ard Biesheuvel and Jason Donenfeld) had adopted KUnit testing, added documentation, simplified how the architecture-optimized code is integrated, and enabled optimizations by default. He had migrated many algorithms into the libraries, and had re-implemented algorithms used by the traditional crypto API to use the libraries. ""All this has resulted in lots of negative diffs, both in the crypto subsystem and in code that uses the crypto code, as well as performance improvements and even some bugs fixed too."" 

He displayed a list of kernel code that is now using the libraries instead of the crypto API (slide 22) that included `apparmor`, `btrfs`, `dm-verity`, `fscrypt`, `bluetooth`, and many others.

In addition, he said that some new features had been added to the crypto library that had not existed in the traditional API. For instance, he [added](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=64edccea594c>) support for [ML-DSA](<https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf>) verification, along with a function for doing so. Support for SHAKE128 and SHAKE256 extendable-output functions (XOF) was [added](<https://lwn.net/ml/all/20250630143834.2748285-1-stefanb%40linux.ibm.com/>) in 6.19. These did not fit into the traditional API, he said, because it assumes fixed-length values. ""In fact, I think the traditional crypto API actually predates cryptographers coming up with the concept of XOFs.""

Biggers said that he had [added support](<https://lwn.net/ml/all/20250915160819.140019-1-ebiggers@kernel.org/>) for SHA-256 interleaved hashing to accelerate dm-verity and fs-verity. ""Again, that's something that didn't really fit into the traditional crypto API"". He mentioned a few other improvements, such as optimizing CRC for arm64, RISC-V, and x86_64; in one case, he said, ""I actually improved performance by 12,400%"". The performance increase was for CRC64 throughput with 16K messages on one of the latest AMD processors.

With regard to testing and code quality, his priority is correctness. That should be pretty self explanatory, he said. ""Crypto code has to be correct, but even non-cryptographic algorithms like CRCs have to be correct too, since people rely on those for data integrity."" He welcomed optimizations, but they had to be worthwhile and testable, and added that he had introduced 18 KUnit test suites for the crypto library and one for the CRC library in the 7.1 kernel.

One of the advantages of KUnit is that it's easier to run the tests now; the tests are already enabled in several continuous-integration (CI) systems, including [KernelCI](<https://kernelci.org/>). ""This wasn't easily possible before because the traditional crypto API uses a custom testing system."" He would like to reach 100% code coverage; the library is already closer to reaching that goal than the traditional API.

There is still work to do, however; architecture-optimized code poses a problem. He said he had been testing about 50 combinations in QEMU, and was requiring QEMU support for new code as well to ensure that it was actually tested. ""I know that Linux doesn't have this policy from most device drivers, but for crypto code, I do think it's the right policy."" 

#### When?

So, when should those working on kernel code that uses crypto or CRC algorithms use the library functions? ""The answer is basically whenever you can. In most cases, I think you'll find the library functions are clearly better."" There were two exceptions, Biggers said: the first was when the crypto library was missing a function that the traditional API has. He cited AES-GCM as an example. ""I'm working on that.""

The other exception was when a kernel feature allowed user space to specify an arbitrary algorithm from the traditional crypto API by name. ""These features generally have to keep using the traditional API for backwards compatibility.""

He advised that anyone adding new kernel features that use cryptography should choose a specific algorithm or, at most, support a small set of algorithms. Historically, he said, there has been a tendency to ""just support every algorithm by accepting an arbitrary string from user space and passing it to the traditional crypto API"". That was a problem, because it allowed use of algorithms that were insecure, obsolete, or make no sense for the feature in question. ""For example, MD5. And also MD4, just in case you thought MD5 was a bit too cutting edge.""

He called supporting every algorithm in every kernel feature ""a huge footgun for users and a maintenance burden for kernel developers"" that increases the attack surface and causes vulnerabilities. He strongly recommended being thoughtful and opinionated about choosing just one good algorithm for new features. ""Keep in mind that you can always add more later if ever needed. Also, if you need help choosing algorithms, you can reach out to the Linux crypto mailing list.""

#### `AF_ALG`

He said that he wanted to call attention to ""yet another exciting problem"" with the traditional API: the algorithm address family, or `AF_ALG`. It was added 16 years ago, he said, ""and it was a mistake"". It exposes almost all of the traditional crypto API to user-space programs, in a bug-prone way, which has resulted in ""a continuous stream of security vulnerabilities"". What is worse, those vulnerabilities were becoming easier to find and exploit. He did not elaborate on why that was, but the obvious answer is the use of LLM tools to scan for security vulnerabilities.

He said that the audience had probably heard of [Copy Fail](<https://copy.fail/>), but there were an additional four `AF_ALG` bugs in the past year with working privilege-escalation exploits. The good news, he said, was that `AF_ALG` had not been commonly used ""because it does not provide much that can't be done in user space"". That meant that it could be disabled by Linux distributions once a few programs were fixed to use user-space code.

However, Biggers acknowledged that breaking a user-space API in Linux ""is kind of a big deal, and it needs to be a community effort"". He said he was issuing a call for action to help harden and deprecate `AF_ALG`; for example, people could help by migrating user-space programs away from `AF_ALG` or by turning it off on their systems and helping to fix anything that breaks when doing so. On that positive note, he said a bit wryly, it was the end of the talk.

There was one question from the audience about how Biggers actually went about improving performance; did he do it on the assembly language level? He answered that most of the architecture-optimized code and CRC code was written in assembly language, but there were some cases where compiler intrinsics were used instead. With that, the session was out of time.

[I would like to thank the Linux Foundation, LWN's travel sponsor, for supporting my trip to Minneapolis for the Linux Security Summit.]
