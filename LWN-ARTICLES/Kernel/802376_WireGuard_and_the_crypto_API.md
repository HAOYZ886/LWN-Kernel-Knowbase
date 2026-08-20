---
title: WireGuard and the crypto API
url: https://lwn.net/Articles/802376/
date: "October 16, 2019"
category: "Cryptography; Networking-Virtual private networks"
author: "By Jake Edge October 16, 2019"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
October 16, 2019

When last we [looked in](<https://lwn.net/Articles/783973/>) on the progress of the [WireGuard VPN tunnel](<https://www.wireguard.com>) toward the mainline kernel, it seemed like the main sticking point had been overcome. The [Zinc cryptography API](<https://lwn.net/Articles/770750/>) used by WireGuard was generally seen as a duplication of effort with the existing kernel cryptographic algorithms, so an effort to rework Zinc to use that existing code seemed destined to route around that problem and bring WireGuard to the mainline. In the six months since then, though, things have gone fairly quiet in WireGuard-land; that all changed based on a conversation at the recent [Kernel Recipes](<https://kernel-recipes.org/en/2019/>) conference in Paris. 

WireGuard developer Jason A. Donenfeld posted a [message](<https://lwn.net/ml/netdev/CAHmME9pmfZAp5zd9BDLFc2fWUhtzZcjYZc2atTPTyNFFmEdHLg@mail.gmail.com/>) from the conference describing a conversation he had there that included kernel networking maintainer David Miller. In the message, Donenfeld announced that WireGuard would be ported to use the existing crypto API in the interests of getting it upstream—based on Miller's advice. Donenfeld said that he was generally opposed to the idea for a few reasons, but now thinks it would make sense to go that route ""and afterwards work evolutionarily to get Zinc into Linux piecemeal"". He outlined his concerns about the kernel crypto API: 

I've long resisted the idea of porting to the existing crypto API, because I think there are serious problems with it, in terms of primitives, API, performance, and overall safety. I didn't want to ship WireGuard in a form that I thought was sub-optimal from a security perspective, since WireGuard is a security-focused project. 

In the message, though, he apparently overstated Miller's opinions somewhat. Miller [complained](<https://lwn.net/ml/netdev/20190925.113928.2046484827308019751.davem@davemloft.net/>) that he was being misquoted and was concerned about Donenfeld's announcement: 

I didn't say "must" anything, I suggested this as a more [smooth] and efficient way forward. 

I'm also a bit disappointed that you felt the need to so quickly make such an explosive posting to the mailing list when we've just spoken about this amongst ourselves only 20 minutes ago. 

Donenfeld [said](<https://lwn.net/ml/netdev/CAHmME9qK2RWPLWWZGcmtVEHz+vUaVRBNtjv3GutkzWccdogF0w@mail.gmail.com/>) that "explosiveness" was not his intent, but that informing the project and others interested in it was what he was aiming for. It turns out that Ard Biesheuvel, who has been critical of the approach tying WireGuard to Zinc along the way, has been working on a patch series to ""incorporate WireGuard into the kernel without relying on a wholesale replacement of the existing crypto stack"". He [posted](<https://lwn.net/ml/linux-crypto/20190925161255.1871-1-ard.biesheuvel@linaro.org/>) the series to the linux-crypto mailing list—seemingly in response to Donenfeld's announcement. 

Donenfeld's [reply](<https://lwn.net/ml/linux-crypto/CAHmME9oDhnv7aX77oEERof0TGihk4mDe9B_A3AntaTTVsg9aoA@mail.gmail.com/>) was positive overall, but there were still some fairly strong criticisms of the approach. To start with, he is concerned with the performance of using indirect function calls as part of the handshake process: 

In this case, the relevance is that the handshake in WireGuard is extremely performance sensitive, in order to fend off DoS. One of the big design gambits in WireGuard is – can we make it 1-RTT [round-trip time] to reduce the complexity of the state machine, but keep the crypto efficient enough that this is still safe to do from a DoS perspective. The protocol succeeds at this goal, but in many ways, just by a hair when at scale, and so I’m really quite loathe to decrease handshake performance. 

He is also unhappy with the use of the asynchronous-oriented parts of the crypto API, which was a complaint [first raised](<https://lwn.net/ml/linux-crypto/CAHk-=wjYsbxSiV_XKWV3BwGvau_hUvQiQHLOoc7vLUZt0Wqzfw@mail.gmail.com/>) by Linus Torvalds. Both Torvalds and Donenfeld think that avoiding the asynchronous interface is best, at least for the initial merge. Donenfeld said: 

I’d recommend leaving this synchronous as it exists now, as Linus suggested, and we can revisit that later down the road. There are a number of improvements to the async API we could make down the line that could make this viable in WireGuard. 

But he definitely wanted to make it clear that he would like to work with Biesheuvel on getting something ready for the mainline: 

And for the avoidance of doubt, or in case any of the above message belied something different, I really am happy and relieved to have an opportunity to work on this _with you_, and I am much more open than before to compromise and finding practical solutions to the past political issues. 

For his part, Biesheuvel [seems pleased](<https://lwn.net/ml/linux-crypto/CAKv+Gu-RLRhwDahgvfvr2J9R+3GPM6vh4mjO73VcekusdzbuMA@mail.gmail.com/>) to get things moving along again. He believes that the crypto API as a whole could benefit from moving away from the dynamic dispatch approach: 

This is one of the issues in the 'fix it for everyone else as well' category. If we can improve the crypto API to be less susceptible to these issues (e.g., using static calls), everybody benefits. I'd be happy to collaborate on that. 

"Static calls" are a [technique](<https://lwn.net/Articles/767707/>) that turns indirect function calls into fixed jump instructions, which has performance and other benefits. Peter Zijlstra recently [posted](<https://lwn.net/ml/linux-kernel/20191007082708.01393931.1@infradead.org/>) a `static_call()` patch set, which may be getting closer to being merged. 

Biesheuvel was a bit surprised that the handshake timing is so sensitive. He suggested that it was simply a performance issue, rather than a security problem. ""But the security of any VPN protocol worth its salt should not hinge on the performance delta between the reference C code and a version that was optimized for a particular CPU."" Donenfeld, however, is [adamant](<https://lwn.net/ml/linux-crypto/CAHmME9rKFUvsQ6hhsKjxxVSnyNQsTaqBKGABoHibCiCBmfxCOA@mail.gmail.com/>) that the fast primitives are required in order to avoid denial-of-service security problems. Since there are other reasons that algorithmic flexibility is needed for WireGuard (though without the indirect function call overhead), the problem needs to be solved anyway, he said. 

Based on the feedback on that first approach, Biesheuvel came back with [another RFC patch set](<https://lwn.net/ml/linux-crypto/20190929173850.26055-1-ard.biesheuvel%40linaro.org/>). It reworked the use of the crypto API so that Torvalds's and Donenfeld's concerns about the using asynchronous interfaces were addressed: 

Linus has made it abundantly clear that using the abstract AEAD interface is not acceptable for instantiating a transformation that is known at compile time, so I will abandon that approach for the time being. If anyone turns up with appropriate h/w to run WireGuard in async mode, we might revisit this, but for sync s/w algorithms, a concrete library interface is clearly preferred. 

There were few comments on those patches, so Biesheuvel [followed up](<https://lwn.net/ml/linux-crypto/20191002141713.31189-1-ard.biesheuvel@linaro.org/>) with a v2 of that approach. This time, Donenfeld had some concerns about some of the architecture-specific optimized implementations of the [ChaCha20 cipher](<https://en.wikipedia.org/wiki/Salsa20#ChaCha_variant>), particularly with regard to changes made from the Zinc versions. Those were largely hashed out in the thread, but a bigger question surrounded Biesheuvel's switch away from potentially using static calls. In the patch cover letter, he said: 

As it turns out, we don't actually need static calls for this. Instead, we can simply permit weak symbol references to go unresolved between modules (as we already do in the kernel itself, due to the fact that ELF permits it), and have the accelerated code live in separate modules that may not be loadable on certain systems, or be blacklisted by the user. 

Andy Lutomirski [wondered](<https://lwn.net/ml/linux-crypto/BE18E4E0-D4CC-40B9-96E1-C44D25B879D9@amacapital.net/>) about that mechanism; he pointed out that the symbol resolution was module-loading-order dependent, which might lead to unexpected results: 

Won't this have the counterintuitive property that, if you load the modules in the opposite order, the reference won't be re-resolved and performance will silently regress? 

I think it might be better to allow two different modules to export the same symbol but only allow one of them to be loaded. Or use static calls. 

While Biesheuvel [agreed](<https://lwn.net/ml/linux-crypto/CAKv+Gu8u8Oco==YRPSa4mq_eZyUcB_Apj-k_vo=7WvTwCp8k+A@mail.gmail.com/>) that module-loading order should not affect which implementation gets chosen, the fact that static calls are not yet available, and might not be the right approach even if they were, means that he is proceeding without them: 

I have disregarded static calls and weak references for the time being, and I will proceed with an implementation that uses neither. The downside of this is that, e.g., we are forced to load the NEON module on non-NEON capable hardware without calling any of its code, but this is basically the Zinc situation only with the NEON and the generic code living in different modules. 

That led to the most recent [patch posting](<https://lwn.net/ml/linux-crypto/20191007164610.6881-1-ard.biesheuvel@linaro.org/>) as of this writing. It has some tweaks here and there and removes the weak-references-based implementation. As he put it in the changelog: ""Defer using weak references or static calls until the dust around this has settled"". There are some of the expected, minor-sounding comments on the patches, but overall the sense is that this work is close to being ready for merging. The next step is presumably to post it to the linux-kernel mailing list and, if it passes muster there, get it into linux-next by way of the crypto subsystem tree. Donenfeld [seemed satisfied](<https://lwn.net/ml/linux-crypto/CAHmME9r9o9EmDoYVV=bGs9Yici900aE2YLN8z-RahSvnP0+mRg@mail.gmail.com/>) that it is close to being merge-ready, to the point that he listed things that were deferred from the original Zinc library that might be taken up again after the merge. He also suggested that WireGuard itself would be up next: 

WireGuard - things are quasi-ready, so when the time comes, I look forward to submitting that to Dave's net tree as a single boring [PATCH 1/1]. 

It would seem that there are no real barriers to the inclusion of this crypto code, at least so far, and the WireGuard code itself has never really been controversial—quite the reverse in fact. One would guess that "boring" patch posting will still require review and, quite possibly, revision, but the biggest hurdle has always been the crypto code. With luck, that has been cleared at this point—though it also kind of looked that way back in March. Mainline WireGuard seems quite plausible for the 5.6 or 5.7 kernel; many are looking forward to that day.
