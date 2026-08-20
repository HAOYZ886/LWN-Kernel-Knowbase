---
title: "KS2011: Structured error logging"
url: https://lwn.net/Articles/464276/
date: "October 24, 2011"
category: Kernel messages; Messages
author: "By Jonathan Corbet October 24, 2011 2011 Kernel Summit coverage"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 24, 2011

* * *

[2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>) coverage 

The idea that the messages emitted by the kernel should be more structured is hardly new. There would be value in standardized messages: tools would be easier to write, translations would be facilitated, documentation of messages could be written, and so on. But every attempt to improve the kernel's logging has run aground. That may be the fate of an initiative described at the 2011 kernel summit, but, then, something may yet come of it. 

Lennart Poettering and Kay Sievers got up to talk about their "journal" work. Their point is that syslog is getting quite old - it is essentially 1980's technology - and we need a better way to deal with messages. Output that is (only) human readable is not as useful as it could be. 

Traditional syslog has a lot of problems. Messages are untrusted; they can come from anywhere. There are lot of different logging techniques. Parsing log files is a hard problem - tools that do so generally turn into [![\[Kay Sievers and Lennart Poettering\]](https://static.lwn.net/images/conf/2011/lc-europe/KayAndLennart-sm.jpg)](<https://lwn.net/Articles/464312/>) "regex horrors." Kernel messages tend to have associated context which does not make it into the log file and must be reconstructed, unreliably, later on. Time zone information is missing, the log stream can be manipulated by an attacker, the use of UDP for network logging means that messages can be dropped, there is "no sane access control," and more. In response to all these problems, Lennart and Kay have been designing "the journal," which is intended to provide unified logging through the entire software stack. The user-space side of the journal exists now and is integrated with service management; the kernel code is not yet available. 

The user-space code is able to augment kernel messages with associated context information; it can, for example, add the real device name that matches a message emitted from the kernel. Strings like "sda" make no sense when seen in isolation on today's dynamic systems, even one day later. 

The most controversial part of the plan (so far as they were able to present it) was the idea of attaching a 128-bit randomized tag to each message. These tags would have no central management; developers are expected to simply generate a new tag when they create a new message. Tools could then be taught the meaning of specific tags and could quickly recognize relevant messages in the logs. Translators and documentation writers could attach information to specific tags. The tag database would be maintained on the net and run in user space; kernel developers would not have to care about the message tags at all. 

Linus broke in to make the claim that UUIDs (which is essentially what these tags are) have never worked in the past; why, he asked, do people still try to use them? The IDs simply get copied around and reused; that will happen with kernel messages too, he predicted. Besides, kernel messages already have a random bit string attached to them - it's the string that is the message itself (or the format string from which it is created). Lennart's defense, which included the assertion that UUIDs are used successfully in the EFI BIOS interface, seemed to convince few people. From there, the conversation became rather loud and unstructured; those who have complained about the lack of contention in recent kernel summits should be less unhappy this time around. 

Some of the pieces of a structured logging interface already exist, as Alan Cox pointed out; functions like `dev_printk()` and friends already encode associated device information in a set way. But Christoph Hellwig asserted that what users really care about is actions on objects; what should be done is to add a new interface catering to that need and leave `printk()` alone. As it happens, Lennart and Kay propose the addition of a `journalk()` logging function for just that purpose. Output from `printk()` would go to the journal, but structured messages with IDs would be created with `journalk()`. 

It was suggested that what we are really seeing here is a reinvention of tracepoints. Alan added that things have to be done carefully, since these messages, however they are created, become part of the ABI. He also noted that we have a good interface for logging now: netlink. All that is needed is to put everything together. It turns out that using netlink for the journal is part of the plan - but the code does not exist yet. 

At the end of the session, Alan recommended that the developers not really even think about `printk()`; they should find a way to characterize what they really need, then come up with a mechanism to provide it. Kay and Lennart sat down without having had the time to present their full proposal; Kay promised that they would try again next year. 

Next: [Coming to love cgroups](<https://lwn.net/Articles/464288/>)
