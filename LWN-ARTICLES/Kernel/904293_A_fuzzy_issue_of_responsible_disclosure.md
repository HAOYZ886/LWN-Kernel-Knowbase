---
title: A fuzzy issue of responsible disclosure
url: https://lwn.net/Articles/904293/
date: "August 12, 2022"
category: "Development model-Security issues; Filesystems-Fuzzing"
author: "By Jonathan Corbet August 12, 2022"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 12, 2022

Fuzz testing is the process of supplying a program with random inputs and watching to see what breaks; it has been responsible for the identification of vast numbers of bugs in recent years — and the fixing of many of them. Developers generally appreciate bug reports, but they can sometimes be a bit less enthusiastic about a flood of reports from automated fuzzing systems. A recent discussion around filesystem fuzzing highlighted two points of view on whether the current fuzz-testing activity is a good thing. 

Filesystem code must accept input from two different directions. On one side is the system-call interface used by applications to work with files. Any bugs in this interface can have widespread implications ranging from data corruption to exploitable security vulnerabilities. But filesystem code also must deal with the persistent form of the filesystems it manages. On-disk filesystem representations are complex data structures that can become corrupted in a number of ways, ranging from hardware errors or filesystem bugs all the way to deliberate manipulation by an attacker. 

Crashing when presented with a corrupted filesystem image is considered poor form, so filesystem developers generally try to keep that from happening. But it is hard to envision all of the ways in which a filesystem image can go wrong, especially if the corruption is created deliberately by a hostile actor. Many of our filesystems have their roots in a time when malicious filesystem images were not something that most people worried about; as a result, they may not be entirely well prepared for that situation. For this reason, allowing the mounting of untrusted filesystem images is generally seen as a bad idea. 

It is thus not entirely surprising that, when fuzz-testers turn their attention to filesystem images, they tend to find bugs. Wenqing Liu has been doing this type of fuzzing for a while, resulting in the expected series of bug reports and filesystem fixes. One [recent report](<https://bugzilla.kernel.org/show_bug.cgi?id=216283>) from Liu of a bug found in the ext4 filesystem, though, drew some unhappy responses. XFS developer Darrick Wong [started it off](<https://lwn.net/ml/linux-ext4/YuBKMLw6dpERM95F@magnolia/>) with this complaint: 

> If you are going to run some scripted tool to randomly corrupt the filesystem to find failures, then you have an ethical and moral responsibility to do some of the work to narrow down and identify the cause of the failure, not just throw them at someone to do all the work. 

Lukas Czerner [disagreed](<https://lwn.net/ml/linux-ext4/20220727115307.qco6dn2tqqw52pl7@fedora/>), saying that these bugs exist whether or not they are reported by fuzz testers and that reporters have no particular ethical responsibility to debug the problems they find. But Dave Chinner (also an XFS developer) [saw things differently](<https://lwn.net/ml/linux-ext4/20220727232224.GW3600936@dread.disaster.area/>) and made the case that these fuzzing reports are not "responsible disclosure": 

> Public reports like this require immediate work to determine the scope, impact and risk of the problem to decide what needs to be done next. All public disclosure does is start a race and force developers to have to address it immediately. 
> 
> Responsible disclosure gives developers a short window in which they can perform that analysis without fear that somebody might already be actively exploiting the problem discovered by the fuzzer. 

In a recent [documentation patch](<https://lwn.net/ml/linux-fsdevel/165989701674.2495930.14992920103388372437.stgit@magnolia/>), Wong complained about reports from ""Fuzz Kiddiez"", saying that: ""The XFS maintainers' continuing ability to manage these events presents an ongoing risk to the stability of the development process"". 

A relevant question that neither Chinner nor Wong addressed, though, is which problem reports should be subject to this sort of "responsible disclosure" requirement? The nature of the kernel is such that a large portion of its bugs will have security implications if one looks at them hard enough; that is (part of) why kernel developers rarely even try to identify or separate out security fixes. Taken to its extreme, any public bug report could be seen as a failure to disclose responsibly. If that is not the intent, then the reporter of an ext4 filesystem crash is arguably being asked to make a determination that most kernel developers will not bother with. 

Returning to the discussion, ext4 filesystem maintainer Ted Ts'o [didn't think](<https://lwn.net/ml/linux-ext4/YuH4nY6DGodheXoE@mit.edu/>) that this was a matter of responsible disclosure in any case: 

> I don't particularly worry about "responsible disclosure" because I don't consider fuzzed file system crashes to be a particularly serious security concern. There are some crazy container folks who think containers are just as secure(tm) as VM's, and who advocate allowing untrusted containers to mount arbitrary file system images and expect that this not cause the "host" OS to crash or get compromised. Those people are insane(tm), and I don't particularly worry about their use cases. 

Ts'o went on to say that the hostility toward fuzzing reports in the XFS subsystem has caused fuzz testers to stop trying to work with that filesystem. Chinner [vigorously disagreed](<https://lwn.net/ml/linux-ext4/20220802032559.GB3861211@dread.disaster.area/>) with that statement, saying that the lack of fuzz-testing reports for XFS is due, instead, to the inability of current testers to find any bugs in that filesystem. That, he said, is because the fstests suite contains a set of XFS-specific fuzzing tests, so the sorts of bugs that fuzz testers can find have already been fixed in XFS. 

Chinner also challenged Ts'o's description of this kind of bug as being low priority: 

> All it requires is a supply chain to be subverted somewhere, and now the USB drive that contains the drivers for your special hardware from a manufacturer you trust (and with manufacturer trust/anti-tamper seals intact) now powns your machine when you plug it in. 

Ts'o, though, [doubled down](<https://lwn.net/ml/linux-ext4/Yuh4n60F3i%2FKBTTV@mit.edu/>) on the claim that exploiting these bugs requires physical access and said that, if an attacker has that access, there are many other bad things that can happen. Attackers have fuzzers too and know how to run them, he added, so little is gained by keeping the results hidden. 

As one might imagine, there was no meeting of the minds that brought this exchange to a happy conclusion. Little is likely to change in any case; the people actually doing the fuzz testing were not a part of the conversation, and would be unlikely to change their behavior even if they had been. There appears to be a strong incentive to run up counts of bugs found by automated systems; it is not surprising that people respond to those incentives by developing and running those systems — and publicly posting the results. 

The best solution may well be _not_ doing as the XFS developers say (keeping crash reports hidden until developers agree that they can be disclosed) but, instead, as the XFS developers do. As Chinner described it, keeping the fuzzing tests in fstests happy has resulted in XFS ""becoming largely immune to randomised fuzzing techniques"". This protection clearly cannot be absolute; otherwise the XFS developers would view the activities of fuzz testers with more equanimity. But it may be indicative of the best way forward. 

Making filesystems robust in the face of corrupted images has often come second to priorities like adding features and improving performance, but the experience with XFS would seem to indicate that, with some focused effort, progress can be made in that direction. Increasing the energy put into solidifying that side of filesystem code could make the issue of responsible disclosure of filesystem-image problems nearly moot.
