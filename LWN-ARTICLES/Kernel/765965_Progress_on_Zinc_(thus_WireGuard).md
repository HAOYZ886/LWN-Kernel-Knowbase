---
title: Progress on Zinc (thus WireGuard)
url: https://lwn.net/Articles/765965/
date: "September 26, 2018"
category: "Cryptography; Network-Virtual private networks"
author: "By Jake Edge September 26, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
September 26, 2018

When last we [looked](<https://lwn.net/Articles/761939/>) at the [WireGuard VPN](<https://www.wireguard.com/>) code and its progress toward mainline inclusion, said progress was impeded by disagreements about the new "Zinc" cryptographic library that is added by the WireGuard patches. Since that August look, several more versions of WireGuard and Zinc have been posted; it would seem that Zinc is getting closer to being accepted. Once that happens, the networking developers are poised to review that portion of the code, which likely will lead to WireGuard in the kernel some time in the next development cycle or two. 

Jason Donenfeld [posted](<https://lwn.net/ml/linux-kernel/20180911010838.8818-3-Jason@zx2c4.com/>) Zinc v3 as part of an updated WireGuard posting on September 10\. Of the versions he has posted since our article (up to v6 as of this writing), v3 has gotten most of the comments. One of the main complaints about Zinc is that it creates a new crypto API in the kernel without really addressing why the existing one would not work for WireGuard. As Ard Biesheuvel [put it](<https://lwn.net/ml/linux-kernel/CAKv+Gu9nSeo4PFVrtjtqGQ32eqyxUp3xVCczZfu+dhFz5yLM0A@mail.gmail.com/>): 

In spite of the wall of text, you fail to point out exactly why the existing AEAD [authenticated encryption with associated data] API in unsuitable, and why fixing it is not an option. 

As I pointed out in a previous version, I don't think we need a separate crypto API/library in the kernel, and I don't think you have convinced anyone else yet either. 

Perhaps you can devote /your/ rare talent and energy to improving what we already have for everybody's sake, rather than providing a completely separate crypto stack that only benefits WireGuard (unless you yourself port the existing crypto API software algorithms to this crypto stack first and present *that* work as a convincing case in itself) 

But Greg Kroah-Hartman [said](<https://lwn.net/ml/linux-kernel/20180911145624.GA21635@kroah.com/>) that the current crypto API is too hard to use for many parts of the kernel, which leads to simpler, private implementations of crypto primitives sprinkled all over the kernel tree. He suggested that the existing crypto API be switched over time to use the Zinc primitives where that is possible. But Eric Biggers was [concerned](<https://lwn.net/ml/linux-kernel/20180911214737.GA81235@gmail.com/>) that no conversions of that sort have been done, which means there could be undiscovered problems in the Zinc API that will make it difficult to do so: ""I don't think it makes sense to merge all this stuff without doing the conversions, or at the very least demonstrating how they will be done"". 

Donenfeld [said](<https://lwn.net/ml/linux-kernel/CAHmME9rFUruF-VN1pmU-k5nFsb9ppAPhPpW-5Ho9dKL2HCg4kA@mail.gmail.com/>) that he is willing to do those conversions, but wants to get the series merged first. ""I'd really prefer to land this series through net-next, and then after we can turn our attention to integrating this into the existing crypto API"". But, as Andrew Lunn [pointed out](<https://lwn.net/ml/linux-kernel/20180911233015.GD11474@lunn.ch/>), that may be putting the cart ahead of the horse. He noted that the networking developers have not had a serious look at the WireGuard patches and won't ""until the controversial part of the code, Zinc, has been sorted out"". He also predicted that networking maintainer David Miller would not take the code into his tree without an Acked-by from the crypto maintainers. 

Miller [confirmed](<https://lwn.net/ml/linux-kernel/20180911.165739.2032677219588723041.davem@davemloft.net/>) that assessment and [clarified](<https://lwn.net/ml/linux-kernel/20180916.214549.234387075817326903.davem@davemloft.net/>) that even though he is listed as one of the two crypto maintainers, he would be looking for an ack from the other maintainer, Herbert Xu, as ""I haven't done a serious review of crypto code in ages"". Xu has been quiet, so far, on the Zinc patches, with one exception. 

Donenfeld feels that Biesheuvel is not pleased with the existence of Zinc. When Biesheuvel [listed](<https://lwn.net/ml/linux-kernel/CAKv+Gu8QgRo-Oex2Sk5unET3FMq+1Cp2btAWXCB8xsALxjatHg@mail.gmail.com/>) the additional items that he thinks need to be addressed in Zinc, Donenfeld's [response](<https://lwn.net/ml/linux-kernel/CAHmME9qwRzuoo-3Hxahwu=Li2LCz06Uowaq1GFmkts6tsffM7w@mail.gmail.com/>) is prefaced by a number of worries about Biesheuvel: that he is ""generally hostile to this whole initiative"", is trying to ""stall it indefinitely"", and perhaps will just keep bikeshedding ""until Zinc copies lots of the same design decisions from the present crypto API"". Donenfeld did also say that he hoped these were all just fears and did not truly reflect what was happening. But that was when Xu stepped in to make it clear that [he values the review](<https://lwn.net/ml/linux-kernel/20180918042133.4ybztgdooxkoyiru@gondor.apana.org.au/>) that Biesheuvel has been doing: 

That may be your view but from what I've read Ard has been very constructive in pointing out issues in your submission. If your response to criticism is to dismiss them as hostile then I'm afraid that we will not be able to progress on this patch series. 

Please keep in mind that this is a large project that has to support multiple users on one hand (not just WireGuard) and complex hardware acceleration drivers on the other. Ard has been one of the most prolific contributors to the crypto code and his review should be taken seriously. 

For his part, Biesheuvel tried to [make his intentions clear](<https://lwn.net/ml/linux-kernel/CAKv+Gu9GVbSETjp01tANMwJgA6O9aexhnH+47836KjZg+71q2A@mail.gmail.com/>) to Donenfeld in another part of the thread: 

I am not an 'entrenched crypto API guy that is out to get you'. I am a core ARM developer that takes an interest in crypto, shares your concern about the usability of the crypto API, and tries to ensure that what we end up is maintainable and usable for everybody. 

But the main technical objections that Biggers and Biesheuvel have raised were still being hashed out in the thread. Andy Lutomirski [suggested](<https://lwn.net/ml/linux-kernel/49BAF465-B3DC-4155-BFF9-DB6C386C1878@amacapital.net/>) that Donenfeld add a conversion of one of the algorithms in the existing crypto API to use Zinc as part of the patch set. After a bit of resistance, Donenfeld [agreed](<https://lwn.net/ml/linux-kernel/CAHmME9om2rJONj6mMvSPHjiZDQKe05BCe=GonHN=c2Rg4n=pkg@mail.gmail.com/>). Conversions of the Poly1305 hash and ChaCha20 cipher (which are what WireGuard uses) in the existing crypto subsystem were part of the [WireGuard v4 patch set](<https://lwn.net/ml/linux-kernel/20180914162240.7925-1-Jason@zx2c4.com/>). 

Along the way, there have also been discussions about the OpenSSL implementations of some of the primitives (i.e. [CRYPTOGAMS](<https://github.com/dot-asm/cryptogams>)) that were incorporated into Zinc. These are written in assembly, but are actually generated from Perl scripts. Donenfeld has modified the assembly output in order to make it comply with kernel coding standards but has also made some other adjustments. That makes it difficult to keep the implementation in sync with any changes that OpenSSL might make, Biesheuvel [noted](<https://lwn.net/ml/linux-kernel/CAKv+Gu-u7vPkU49AzToGHyf_cWp0MiOmpD3cRUt8260QC2wj-Q@mail.gmail.com/>). ""Dumping 10,000s of lines of generated assembler in the kernel tree like that is really unacceptable IMO."" Donenfeld [said](<https://lwn.net/ml/linux-kernel/20180825170629.GA8971@zx2c4.com/>) that he disagrees with the characterization of the code, but that getting his changes into the OpenSSL upstream is desirable. 

Beyond that, there were some minor licensing concerns (and the resulting [SPDX identifiers](<https://lwn.net/Articles/739183/>)) with some files, which have seemingly been resolved. Similarly, some performance problems were noted and addressed. In short, Zinc is starting to look like something that could be merged. Donenfeld [posted](<https://lwn.net/ml/linux-kernel/20180925145622.29959-1-Jason@zx2c4.com/>) v6 of the WireGuard patch set on September 25\. 

Zinc is still awaiting an ack from Xu, though it is not clear how much he has scrutinized the code at this point. Once that happens, though, the networking side of the patch set can be reviewed by Miller and other networking developers. If all goes well, it will end up in the mainline before too long—but that still means at least four, or more likely seven, months from now. Whenever it comes, it is clear that WireGuard is eagerly anticipated by many.
