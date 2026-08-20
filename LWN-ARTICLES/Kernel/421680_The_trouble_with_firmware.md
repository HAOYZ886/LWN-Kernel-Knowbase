---
title: The trouble with firmware
url: https://lwn.net/Articles/421680/
date: "January 5, 2011"
category: Copyright issues; Firmware
author: "By Jake Edge January 5, 2011"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
January 5, 2011

Within the various communities that make up the free software world, there are many different opinions on exactly what constitutes "freedom". Those opinions run the gamut from extremely pragmatic to exceedingly strict definitions of "free". Firmware blobs for drivers--almost always released without source code and often without any clear license--have been particularly troublesome for certain projects. Debian has [struggled](<https://lwn.net/Articles/304925/>) with the problem over the years, finally resolving it by moving the non-free firmware out of its main repository for the upcoming 6.0 ("Squeeze") release. But there are others who find even that insufficient and would like to see any mention of the non-free firmware files removed from the kernel. 

The [Linux-libre project](<http://www.fsfla.org/svnwiki/selibre/linux-libre/>) aims to deliver a completely free (under its definition) Linux distribution. It does that by removing any programs that either don't come with source code, or that has source code that is licensed in a way that does not allow users to modify it. Many Linux distributions already follow those guidelines, but Linux-libre takes things one step further and removes anything that ""induces or requires you to install additional pieces of non-Free Software"". 

As might be guessed, Linux-libre has a rather liberal interpretation of what "induces" means, which is where the kernel firmware loading mechanism runs afoul of the distribution's goals. Linux-libre has already eliminated the non-free firmware from its repositories, but there is still a problem: the kernel will make a request to user space for (potentially) non-free firmware when it loads a driver that requires it. So in the minds of the Linux-libre developers, the kernel is now inducing users to install non-free software. 

If user space cannot find the firmware file, then the driver obviously cannot load it into the hardware. That makes the hardware non-functional, presumably, but far worse from an inducement perspective, the kernel puts a message into the log specifically noting the filename that couldn't be found. Evidently, that will induce users to go find a file by that name and install it, thus resulting in a non-free system. 

In a discussion about Debian's Squeeze kernel on the linux-libre mailing list, Richard Stallman [mentioned](<https://lwn.net/Articles/420334/>) a plan to obfuscate the names of the firmware files within the Linux-libre kernel. Part of his message was picked up as a Kernel page "quote of the week", partly because it seems like something of a contradiction to promote freedom by hiding the proper names of firmware files. It's also a little puzzling because the GPL and the software freedom movement is so clearly focused on the rights of users. 

But Stallman isn't trying to stop users from finding the firmware, he just wants to ensure that the distribution doesn't induce them to install it. That distinction may seem rather arbitrary to many, but not to him. In response to a [concern](<https://lwn.net/Articles/421701/>) expressed by Alexandre Oliva about web pages that would show the mapping between the Linux-libre obfuscated names and those of the actual filenames, Stallman [explained](<https://lwn.net/Articles/421699/>): 

Remember, the goal is not to prevent people from finding these files. [...] Our goal is to avoid steering users towards nonfree software. If they find these identifiers and look them up, I think it will be clear to them that we are not. 

The plan seems to go far beyond just creating a kernel with obfuscated firmware names, though. It would be fairly straightforward to run a script on the kernel source that applied some kind of obfuscation filter to the firmware filenames before building the kernel, but that doesn't completely alleviate the problem for Linux-libre. It would also like to have a kernel Git repository with the mangled filenames, but there are problems with that approach as Oliva [notes](<https://lwn.net/Articles/421714/>): 

I'd love to have this in a git repository rather than as deblobbing scripts, but a solution for the problem of creating a git repository that can track Linux upstream without carrying the non-Free bits that the Linux git repository carries has so far eluded me. 

From there, the conversation starts to take on some Orwellian tones. It is not enough to just have the new firmware filenames in the HEAD of the Git repository, Oliva and Stallman would like to eliminate references to the original filenames (and the firmware files themselves) from the repository _history_. But they also would like to be able to pull from Linus Torvalds's mainline tree periodically. It is a tricky set of conditions to try to fulfill. 

As part of the discussion on how to handle it, Stallman seems to [bristle](<https://lwn.net/Articles/421719/>) over the term "rewriting history"--perhaps because of its Orwellian overtones. He sees that term as (somehow) pushing Torvalds's views: 

In effect, it claims that "If Torvalds let someone put nonfree software into a present or even past version of Linux, everyone who wants to refer to a repository of Linux is obligated to refer to that nonfree code." That is his views. We believe it is wrong to tell anyone about this nonfree software, and we will not follow his views. 

Stallman doesn't necessarily see being able to pull from the mainline as much more than a convenience. But Oliva, who clearly has more Git and kernel workflow experience than Stallman, is fairly [adamant](<https://lwn.net/Articles/421726/>): 

However, it is an absolute must for any of us to be able to trivially pull changes from upstream. Actually, make that upstreams, for any branch that tracks Linus' tree and has additional patches might be desirable for any of us to merge into our personal or published repositories. If the Linux-libre repository doesn't fit into people's regular workflow, there will be a strong incentive against using it, against developing improvements for Linux on it. It would be self-defeating. I don't want this kind of pain, not for myself, not for other contributors. 

To that end, Oliva [asked](<https://lwn.net/Articles/421729/>) for ways to accomplish this goal on the Git mailing list. He comes up with some plausible scenarios where others might want this kind of functionality, beyond just the needs of Linux-libre: 

Say, portions that are illegal, immoral or too risky in my jurisdiction: patented stuff that lawyers say I should not distribute in any way, unauthorized or otherwise copyright-infringing bits, text or pictures that are offensive or even illegal to publish, i.e., stuff that I must not be caught distributing and that, ideally, I could arrange to not even possess. 

A few suggestions of possible workflows were made in that thread, but Oliva seems to be [leaning toward](<https://lwn.net/Articles/421730/>) using `git-filter-branch` and either some scripts or changes to Git to facilitate this effort. While his scenarios are a little bit of a stretch--why distribute a project that contains these problematic pieces?--they aren't completely beyond the pale. Others could benefit from whatever solution Linux-libre finally settles on. 

It's certainly not the first time that Stallman has seemed to be tilting at windmills, and plenty of folks will find this effort to be pointless, even comical. There are some good arguments on both sides of the firmware debate, but it is a bit hard to see how making it _more_ difficult to get hardware working on Linux-libre is going to lead to more adoption of that distribution. Far more likely, it would seem, is that folks just grab a different distribution (or operating system entirely) and move on.
