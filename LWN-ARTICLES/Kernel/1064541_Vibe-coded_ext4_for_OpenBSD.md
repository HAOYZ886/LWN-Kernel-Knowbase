---
title: "Vibe-coded ext4 for OpenBSD"
url: https://lwn.net/Articles/1064541/
date: "March 26, 2026"
category: "Filesystems-ext4"
author: "By Jonathan Corbet March 26, 2026"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 26, 2026

A number of projects have been struggling with the question of which submissions created by large language models (LLMs), if any, should be accepted into their code base. This discussion has been further muddied by efforts to use LLM-driven reimplemention as a way to remove copyleft restrictions from a body of existing code, as recently [happened with the Python chardet module](<https://lwn.net/Articles/1061534/>). In this context, an attempt to introduce an LLM-generated implementation of the Linux ext4 filesystem into OpenBSD was always going to create some fireworks, but that project has its own, clearly defined reasons for looking askance at such submissions. 

It all started on March 17, when Thomas de Grivel posted [an ext4 implementation](<https://marc.info/?l=openbsd-tech&m=177393091219373&w=2>) to the openbsd-tech mailing list. This implementation, he said, provides full read and write access and passes the `e2fsck` filesystem checker; it does not support journaling, however. The code includes a number of copyright assertions, but says nothing about how it was written. In [a blog post](<https://www.kmx.io/blog/openbsd-ext4fs-update>), though, de Grivel was more forthcoming about the code's provenance: 

> No Linux source files were ever read to build this driver. It's pure AI (ChatGPT and Claude-code) and careful code reviews and error checking and building kernel and rebooting/testing from my part. 

There were a number of predictable concerns raised about this code, many having to do with the possibility that it could be considered to be a derived product of the (GPL-licensed) Linux implementation. The fact that the LLM in question was almost certainly trained on the Linux ext4 code and documentation does not help. Bringing GPL-licensed code into OpenBSD is, to put it lightly, not appreciated; Christian Schulte [was concerned](<https://marc.info/?l=openbsd-tech&m=177424913926828&w=2>) about license contamination: 

> I searched for documentation about that ext4 filesystem in question. I found some GPL licensed wiki pages. The majority of available documentation either directly or indirectly points at GPL licensed code. In my understanding of the issue discussed in this thread this already introduces licensing issues. Even if you would write an ext4 filesystem driver from scratch for base, you would almost always need to incorporate knowledge carrying an illiberal license. 

Theo de Raadt, however, [pointed out](<https://marc.info/?l=openbsd-tech&m=177424963227139&w=2>) that reimplementation of structures and algorithms is allowed by copyright law; that is how interoperability happens. One should not conclude that De Raadt was in favor of merging this contribution, though. 

From the OpenBSD point of view, the copyright status of LLM-generated code is indeed problematic, for the simple reason that nobody knows what that status is, or even if a copyright can exist on that code at all. Without copyright, it is not possible to grant the project the rights it needs to redistribute the code. As De Raadt [explained](<https://marc.info/?l=openbsd-tech&m=177411863202734&w=2>): 

> At present, the software community and the legal community are unwilling to accept that the product of a (commercial, hah) AI system produces is Copyrightable by the person who merely directed the AI. 
> 
> And the AI, or AI companies, are not recognized as being able to do this under Copyright treaties or laws, either. Even before we get to the point that the AI's are corpus-blenders and Copyright-blenders. 
> 
> So as of today, the Copyright system does not have a way for the output of a non-human produced set of files to contain the grant of permissions which the OpenBSD project needs to perform combination and redistribution. 

Damien Miller [said something similar](<https://marc.info/?l=openbsd-tech&m=177423545420236&w=2>): 

> Who is the copyright holder in this case? It clearly draws heavily from an existing work, and it's clear the human offering the patch didn't do it. It's not the AI, because only persons can own copyright. Is it the set of people whose work was represented in the training corpus? Was the it the set of people who wrote ext4 and whose work was in the training corpus? The company who own the AI who wrote the code? Someone else? 
> 
> We don't know. The law hasn't caught up to the technology yet and we can't take the risk that, when it does, it will go in a way that makes use of AI-written code now expose us to legal risk. 

These words did not resonate entirely well with de Grivel, who [refused](<https://marc.info/?l=openbsd-tech&m=177423717520934&w=2>) to retract his copyright claims on the machine-generated code. He also [is clearly pleased](<https://marc.info/?l=openbsd-tech&m=177412083903778&w=2>) with the kinds of things one can do with LLMs: 

> We can freely steal each other in a new original way without copyright infringment its totally crazy the amount of code you can steal in just 1h. What took 20 years to Bell labs can now be done in 20 hours straight. 

The conversation went on for some time, but the result was never really in doubt; De Raadt [made it clear](<https://marc.info/?l=openbsd-tech&m=177411620801633&w=2>) when he said: ""the chances of us accepting such new code with such a suspicious Copyright situation is zero"". In the above-mentioned blog post, de Grivel added a note on March 23 that he would respond by removing all of the LLM-generated code, leaving only code that he has written himself. After this episode, though, convincing others that he really did write any subsequent versions on his own may be an uphill battle. He [acknowledged](<https://marc.info/?l=openbsd-tech&m=177427012912573&w=2>) that ""forking OpenBSD"" might be easier. 

The number of people who have concluded that they can have an LLM crank out thousands of lines of code and submit the result to the project of their choice is growing quickly. Needless to say, these people are not always diligent about documenting the provenance of the work they are submitting in their own names. There may well come a time when it turns out that even the sharp eyes of OpenBSD reviewers are unable to keep all of it out of their repositories. 

All of this code is setting some worrisome potential traps for the future. As Tyler Anderson [pointed out](<https://marc.info/?l=openbsd-tech&m=177430829313972&w=2>), the price of these tools is unlikely to go down as development projects become more dependent on them. Who will maintain this code, when its original "author" does not understand it and has no personal investment in it, is unclear at best. And if there is, in fact, a potential copyright problem inherent in this code, there will have to be a lot of scrambling (or worse) when it comes to light. Given all of that, it is unsurprising that many projects, especially those with longer time horizons, are proving reluctant to accept machine-generated submissions.
