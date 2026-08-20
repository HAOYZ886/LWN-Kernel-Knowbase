---
title: Reconsidering Speck
url: https://lwn.net/Articles/761992/
date: "August 8, 2018"
category: Cryptography
author: "By Jake Edge August 8, 2018"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
August 8, 2018

The [Speck cipher](<https://en.wikipedia.org/wiki/Speck_\(cipher\)>) is geared toward good performance in software, which makes it attractive for smaller, often embedded, systems with underpowered CPUs that lack hardware crypto acceleration. But it also comes from the US National Security Agency (NSA), which worries lots of people outside the US—and, in truth, a fair number of US citizens as well. The NSA has earned a reputation for promulgating various types of cryptographic algorithms with dubious properties. While the technical arguments against Speck, which is a fairly simple and straightforward algorithm with little room for backdoors, have not been all that compelling, the political arguments are potent—to the point where it is being dropped by the main proponent for including it in the kernel. 

#### A bit of history

Speck was merged for the 4.17 kernel and the fscrypt module for ext4 and F2FS added Speck128 and Speck256 support in 4.18. Speck is a block cipher, rather than a stream cipher, which makes it suitable for uses like filesystem encryption. As Eric Biggers [noted](<https://www.spinics.net/lists/linux-crypto/msg31418.html>) when Speck was proposed in February, it is a good choice for low-end CPUs: 

Speck has been somewhat controversial due to its origin. Nevertheless, it has a straightforward design (it's an ARX cipher), and it appears to be the leading software-optimized lightweight block cipher currently, with the most cryptanalysis. It's also easy to implement without side channels, unlike AES. Moreover, we only intend Speck to be used when the status quo is no encryption, due to AES not being fast enough. 

But the "controversial" nature of Speck that he referred to soon reared its head. In response to version 2 of the patch set in April, Jason A. Donenfeld [questioned the move](<https://www.spinics.net/lists/linux-crypto/msg32882.html>): ""Can we please not Speck?"" He noted that Speck (and its hardware-oriented counterpart, [Simon](<https://en.wikipedia.org/wiki/Simon_\(cipher\)>)) had recently been [rejected by ISO](<https://www.schneier.com/blog/archives/2018/04/two_nsa_algorit.html>). Biggers [acknowledged Donenfeld's complaint](<https://www.spinics.net/lists/linux-crypto/msg32887.html>), but asked what alternative he would suggest. Furthermore: 

As I explained in the patch, the purpose of adding Speck is to allow low-end Android devices -- ones that have CPUs without the ARMv8 Cryptography Extensions \-- to start using dm-crypt or fscrypt. Currently such devices are unencrypted. So, Speck is replacing *no encryption*, not another encryption algorithm. By removing Speck, you are removing encryption. It's great that people are enthusiastic about debating choices of crypto algorithms. But it's unfortunate that "no crypto" tends to pass by without comment from the same people. 

The ISO rejection was based on NSA refusal to answer questions about Speck and Simon, particularly with regard to what cryptanalysis the agency had already done on them, [according to Tomer Ashur](<https://www.spinics.net/lists/linux-crypto/msg33291.html>), who was part of the ISO group that rejected the ciphers. In that lengthy message, which came a few months after the rest of the discussion, Ashur outlined a number of different problems that he and others see with Speck and the NSA's behavior—though no serious technical flaws have been found in the algorithm itself. 

Donenfeld said that one of his concerns was that ""some of the best symmetric cryptographers in academia have expressed reservations about it"", but did not offer up any alternative that might fit the bill. Biggers had mentioned some work that Google has done on alternatives, but there were concerns there as well: 

Paul Crowley actually designed a very neat wide-block encryption mode based on ChaCha20 and Poly1305, which we considered too. But it would have been harder to implement, and we'd have had to be pushing it with zero or very little outside crypto review, vs. the many cryptanalysis papers on Speck. (In that respect the controversy about Speck has actually become an advantage, as it has received much more cryptanalysis than other lightweight block ciphers.) 

Samuel Neves did [have some suggestions](<https://www.spinics.net/lists/linux-crypto/msg32900.html>) on alternatives, however. He listed a handful of ciphers that might be worth investigating; Biggers implemented and compared many of those in a [post](<https://www.spinics.net/lists/linux-crypto/msg33000.html>) in early May. The other algorithms were mostly slower than Speck and those that weren't suffered from other shortcomings. In that message, he mentioned Crowley's work again, with an eye toward proposing it as an alternative at some point: 

Still, we don't want to abandon HPolyC (Paul's new ChaCha and Poly1305-based wide-block mode), and eventually we hope to offer it as an option as well. But it's not yet published, and it's a more complex algorithm that is harder to implement so I haven't yet had a chance to implement and benchmark it. And we don't want to continue to leave users unprotected while we spend a long time coming up with the perfect algorithm, or for hardware AES support to arrive to all low-end CPUs when it's unclear if/when that will happen. 

#### Android dropping Speck

Since then, Google has decided not to use Speck and to pursue HPolyC (which is described in [this paper [PDF]](<https://eprint.iacr.org/2018/720.pdf>)), Biggers said in an [RFC patch set](<https://lwn.net/ml/linux-crypto/20180806223300.113891-1-ebiggers@kernel.org/>) that was posted August 6\. The patch set implements primitives for [XChaCha20](<https://download.libsodium.org/doc/advanced/xchacha20.html>), XChaCha12 (which has fewer rounds), and the [Poly1305](<https://en.wikipedia.org/wiki/Poly1305>) cryptographic hash for the Linux crypto subsystem. HPolyC is a combination of those primitives: 

HPolyC encrypts each message using XChaCha12 or XChaCha20 sandwiched between two passes of Poly1305, plus a single block cipher invocation (e.g. AES-256) per message. On ARM Cortex-A7, on 4096-byte messages HPolyC-XChaCha12-AES is slightly faster than Speck128/256-XTS. Note that for long messages, the block cipher is not performance-critical since it's only invoked once per message; that's why we can use AES in HPolyC, despite the fully AES-based encryption modes being too slow. 

HPolyC is a construction, not a primitive. It is proven secure if XChaCha and AES are secure, subject to a security bound. Unless there is a mistake in this proof, one therefore does not need to trust HPolyC; one need only trust XChaCha (which itself has a security reduction to ChaCha) and AES. 

The switch to 12 rounds for ChaCha, from the more usual 20, was [questioned](<https://lwn.net/ml/linux-crypto/CAHmME9rbe6eT6dkUqKnp-GxcqGrp-tvMfTMr8kZtsn=S3cmP8A@mail.gmail.com/>) by Donenfeld. Though he believes ChaCha12 ""probably still provides adequate security"", he is concerned that ""introducing ChaCha12 into the ecosystem feels like a bit of a step backwards"". He wondered what testing had been done to determine that 12 rounds was needed instead of 20. 

Crowley [pointed out](<https://lwn.net/ml/linux-crypto/CA+_SqcD+KpQ_X6iXUCpQh0N6pWzBorstBJDx7eBqcybY4Dvy8Q@mail.gmail.com/>) that the best attack on ChaCha can only break seven rounds and requires 2248 operations to do so. ""Every round of ChaCha makes attacks vastly harder."" Neves [agreed](<https://lwn.net/ml/linux-crypto/CAEX_ruFp0KddnZyetZt997OvRzKDuN7U95MF_N9AgTh+mTcD7Q@mail.gmail.com/>) that 12 rounds was reasonable, but did note that more recent attacks on ChaCha7 have reduced the complexity to 2235: 

In any case, every attack so far appears to hit a wall at 8 rounds, with 12 rounds---the recommended eSTREAM round number for Salsa20---seeming to offer a reasonable security margin, still somewhat better than that of the AES. 

Meanwhile, Crowley [said](<https://lwn.net/ml/linux-crypto/CA+_SqcBS-xi7xXK0BFW5UneQ6ocqZKLuUHCaVTMTqMpj4Wu7Pg@mail.gmail.com/>) that the performance of HPolyC is ""still a lot slower than I'm happy with, and encryption still has a quite noticeable effect on the feel of low end devices"" even using ChaCha12. Since it [provides](<https://lwn.net/ml/linux-crypto/CA+_SqcD+KpQ_X6iXUCpQh0N6pWzBorstBJDx7eBqcybY4Dvy8Q@mail.gmail.com/>) ""a solid margin of security"", ChaCha12 is what was chosen. He also noted that, even if all handsets were to get accelerated AES at some point, the low-end problem doesn't go away: ""we'll probably be worrying about it for IoT devices"". 

#### Remove Speck from the kernel?

Since Google is no longer planning to use Speck, Donenfeld [posted a patch](<https://lwn.net/ml/linux-crypto/20180806230437.21431-1-Jason@zx2c4.com/>) to remove Speck from the kernel. Biggers [was not opposed](<https://lwn.net/ml/linux-crypto/20180807011937.GA133621@gmail.com/>) and acked the patch, though he did want to clarify that there were no technical flaws that he (or Google) knows about in Speck. There are other things to take into account, he said: 

However, clearly today there are more than just technical considerations when choosing cryptographic primitives. So ultimately, enough people didn't *want* Speck that we weren't able to offer it, even though it was only meant to replace no encryption. 

Jeffrey Walton [argued](<https://lwn.net/ml/linux-crypto/CAH8yC8kKet=fx0rYF8QEWK+TzyBgwHrLabWE72znBY=QLZVWkA@mail.gmail.com/>) against removing Speck in order to provide more algorithm choices. But, as Biggers [pointed out](<https://lwn.net/ml/linux-crypto/20180807201819.GA25300@gmail.com/>), the kernel is probably not the right place to provide that choice: 

The purpose of the Linux kernel's crypto API is to allow kernel code to do crypto, and also sometimes to allow access to crypto accelerator hardware. It's *not* to provide a comprehensive collection of algorithms for userspace programs to use, or to provide reference implementations for crypto algorithms. Before our change in plans, we needed Speck-XTS in the kernel so that it could be used in dm-crypt and fscrypt, which are kernel features and therefore require in-kernel implementations. And of course, our proposed new solution, HPolyC, will need to be supported in the kernel too for the same reason. It's just the way transparent disk and file encryption works; the crypto needs to be done in the kernel. 

But Theodore Y. Ts'o [said](<https://lwn.net/ml/linux-crypto/20180807031526.GD5048@thunk.org/>) that any decision not to use Speck and/or to remove it from the kernel is ""purely political --- not [technical]"". On the other hand, Ard Biesheuvel [sees](<https://lwn.net/ml/linux-crypto/CAKv+Gu_Dr1KvEOMdkhxZtq093MDgvcP_G-5=JMmqsx+zYV-=mw@mail.gmail.com/>) the decision to remove it from the kernel in more pragmatic terms: 

Whether or not to use it may be a political rather than a technical motivation. But the reason I acked this patch is not because removing it aligns with my political conviction regarding Speck, but simply because its contributor, primary intended user and therefore de facto maintainer stated publicly that it no longer had any intention to use it going forward. 

The Speck code is a recent addition to the kernel and, as far as anyone knows, is unused since it will not be appearing in Android handsets. Assuming no other users materialize, it would seem likely that it will be gone before long. While the complaints of Ashur and other cryptographers are, in part, technical, those arguments are not particularly compelling, at least within the kernel community. But a lack of users—and maintainers—for the cipher is a good reason to remove it. While politics may have led to that outcome, it is a reasonable technical argument for its removal. 

The NSA clearly burned many bridges with the cryptography community with its [Dual_EC_DRBG shenanigans](<https://blog.cryptographyengineering.com/2015/01/14/hopefully-last-post-ill-ever-write-on/>) and other actions over the years. It should come as no surprise to the agency or anyone else that cryptographic contributions from the NSA are going to be heavily scrutinized. The likelihood that Speck is backdoored in some way is generally seen as quite low, but being uncooperative during the ISO review is not the way to get out of the hole it has dug for itself. The NSA has a large and potent stable of cryptographers, but its aims are not necessarily aligned with anyone outside its walls, so it is not surprising to see skepticism—or outright rejection—of algorithms it is pushing.
