---
title: Trust in and maintenance of filesystems
url: https://lwn.net/Articles/951846/
date: "November 21, 2023"
category: "Filesystems-Deprecation policy"
author: "By Jonathan Corbet November 21, 2023 Maintainers Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 21, 2023

* * *

[Maintainers Summit](<https://lwn.net/Articles/951847/>)

The Linux kernel supports a wide variety of filesystems, many of which are no longer in heavy use — or, perhaps, any use at all. The kernel code implementing the less-popular filesystems tends to be relatively unpopular as well, receiving little in the way of maintenance. Keeping old filesystems alive does place a burden on kernel developers, though, so it is not surprising that there is pressure to remove the least popular ones. At the 2023 Kernel Maintainers Summit, the developers talked about these filesystems and what can be done about them. 

Christoph Hellwig started the discussion by saying that it is hard for developers to know how mature — how trustworthy and maintained — any given filesystem is; that can often only be determined by talking to its users. This information gap can be a bad thing, he said. User space (in the form of desktop environments in particular) has a strong urge to automatically mount filesystems, even those that are unmaintained, insecure, and untrustworthy. This automounting exposes the system to security threats and is always a bad idea, but it's a fact of life; maybe there needs to be a way for the kernel to indicate to user space that some filesystems are not suitable for mounting in this way. 

[![\[Christoph
Hellwig\]](https://static.lwn.net/images/conf/2023/lpc/ChristophHellwig-sm.png)](<https://lwn.net/Articles/951850/>) Another problem, he said, is fuzz testing. He appreciates all of the work that is going into fuzz-testing of filesystems, but it is not helpful if it is being directed at filesystems that are not going to be fixed. That is a waste of resources; the fuzzer should, instead, be directed at filesystems that will be fixed in response to problem reports. 

The broader question, he continued, is how the kernel can do better at getting rid of old and unmaintained code. The process for doing so is always "very ad hoc"; typically, some maintainer gets angry and sends a removal patch, which is sometimes accepted. What comes next is typically a round of "whining in the Linux press". Other projects have "reasonable deprecation schedules", where features are annotated as being set for removal in a couple of releases unless somebody who cares puts in the time to maintain them properly. The kernel, perhaps, could benefit from something similar. 

He closed by mentioning the prospect of the European [Cyber Resilience Act](<https://lwn.net/Articles/944300/>), which could put vendors of products containing the kernel at risk. 

Steve Rostedt said that, since most users run the kernels provided by distributors, the right thing to do might be to educate those distributors about which filesystems are trustworthy. Hellwig responded that, if the community needs to educate in that way, it is doing something wrong; there should be a better way to communicate this information. Ted Ts'o said that there are hundreds of distributions out there; it would be necessary for the kernel community to decide which ones it cares about. It could be said, he suggested, that distributors that do not contribute back to the kernel do not matter. 

Linus Torvalds responded that he does not want anybody who would make that argument as a developer on his kernel — a position that has not changed in decades, he said. Any approach that says "users don't matter" is wrong, he said. Hellwig answered that users need to know when a filesystem is broken. Dave Chinner suggested that the right approach is to talk to the people who are setting the automounting policies — and the GNOME developers in particular. 

#### Communications

Kees Cook said that the maturity information for filesystems could be stored as a field in the `MAINTAINERS` file; noises in the room made it clear that this idea was not universally loved. Hellwig said that file is not the place to send users; perhaps, instead, the automounter could be taught to only mount filesystems at a given maturity level or above. Chinner noted that the XFS developers have shipped a udev rule saying "don't automount XFS filesystems" for some time; perhaps that policy should be centralized. Torvalds said that everybody agrees that the current "automount everything" policy is wrong, but it should not be blocked in the kernel; this is a problem for the desktop environments. 

Ts'o said that the problem comes down to communications; end users do not read kernel configurations or documentation folders. There is also no agreement on the acceptable level of risk; kernel developers worry about unmaintained filesystems, but GNOME developers think that everything should be automounted. That was a rule made by a product manager, he said, "and you don't argue with product managers". That, in turn, makes his life miserable as he is inundated with syzbot reports for crashes resulting from mounting malicious filesystems. The kernel, he concluded, should suggest a default policy that makes sense, but user space will make the decision. 

If we had some sort of filesystem maturity model, Hellwig said, distributors would be able to use it to set an automounting policy. Josef Bacik said that it was just necessary to figure out a way to communicate this information to user space. Once that is in place, it's just a matter of waiting a couple of years for the tooling to be updated. He added that the more interesting problem was that of deprecation; there is no policy for doing that now. He would like a way to tag kernel features as being deprecated and slated for future removal. Greg Kroah-Hartman pointed out that this idea has been tried in the past (it was [abandoned](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9c0ece069b32>) in 2012); Bacik said that it was time to try again. 

Kroah-Hartman said that the kernel drops drivers all the time, and that perhaps the process should be formalized. Torvalds answered that he will always require a reason to deprecate code; the lack of reasons has annoyed him in the filesystem discussion, he said. He raised the "sysv" filesystem (used on Unix version 7 on PDP-11 machines and on some early proprietary x86 Unix systems) as an example; it is simple and places no burden on developers, so there is no reason to deprecate it. Bacik responded that there is no way to make changes to sysv and be sure of not breaking it; when Torvalds asked if anybody had encountered a problem with sysv, the answer from the room was "we don't know". He said that proved the lack of a cost, so nobody cares if sysv remains in the tree. 

There was a side discussion on the differences between filesystems and drivers that started when Torvalds pointed out that there are many drivers in the kernel that receive little or no testing. Hellwig asserted that there are many drivers that have not worked for years. Kroah-Hartman said that there is no difference between drivers and filesystems, but Arnd Bergmann said that, for drivers, it's relatively clear when the associated hardware no longer exists. That is not the case for filesystems. Thomas Gleixner added that, if a driver stops working, it simply stops; a filesystem, instead, can silently corrupt data. Chinner agreed, pointing out that one cannot fix filesystem corruption with a reboot. 

Rostedt claimed that it is possible to test all of the filesystems in the tree, since there is no special hardware needed, but the filesystem developers disagreed. Chinner said that, without a `mkfs` tool, filesystem testing cannot be done; additionally, there is a need for a filesystem checker and integration with the fstests suite. Ts'o singled out the ntfs filesystem; the tools are proprietary, but the kernel developers took the kernel code without insisting that the tools be open. 

According to Chinner, the quality of the kernel's filesystem implementations has improved greatly over the last decade or so. There are something like 2,000 tests in the fstests suite; it even has 200 Btrfs-specific tests, where recently there were none. The reliability of the filesystems covered by fstests has gone way up; developers know that the filesystems are good and can tell users so. For the other filesystems, nobody really knows. 

Torvalds pointed out that one of those other filesystems, reiserfs, is in fact deprecated and on its way out. It is possible to remove code that causes problems, but only if there is a good reason to do so. Reiserfs may still have a few users, given that SUSE defaulted to it for years, but he is happy to remove it — as long as there are no complaints, which might cause him to reconsider. Brauner asked for a proper path toward that removal, so that maintainers can effect removals without getting yelled at. 

There was some discussion of how to communicate filesystem maturity information to user space. Ideas included an ELF section in module binaries, a kernel-configuration option, an interface for automounters, tainting the kernel when a low-quality filesystem is mounted, or requiring a special "I know it's deprecated" mount option. Dave Airlie suggested working with the [udisks](<https://www.freedesktop.org/wiki/Software/udisks/>) developers; Chinner said that he had tried that and "hit a brick wall" before just adding a udev rule instead. A few developers expressed frustration with a perceived inability to get a response from user-space developers on topics like this. 

#### "Print a warning"

Torvalds said that there is no way to communicate this information to existing user space, since that code is not prepared to receive that message. Instead, he said, just outputting a warning with `printk()` can be effective; users see the errors and complain to their distributors. Suitable warnings could be added to the more questionable filesystems. 

That, Kroah-Hartman said, requires coming up with a list of good and bad filesystems. Hellwig said that there would need to be at least three levels: "no trust", "generally maintained but don't mount untrusted images", and "well maintained". Torvalds said that this information could be given to the kernel when a filesystem is registered, and a warning printed if an untrusted filesystem is mounted. "Enough arguing", he said, it was time to just write a patch and try it. 

The arguing was not done quite yet, though; Gleixner complained about architectures that do not keep up with low-level core-kernel changes. That leaves him having to figure out how to fix "25 PowerPC subarchitectures", an impossible task for a developer without the hardware and who is not an expert in that architecture. Might there be a way to tell architecture maintainers that they need to move forward to current APIs or the support will be removed? 

Hellwig concurred with the problem, saying that there is an implicit assumption in the community that this sort of API cleanup is a low-priority task; as a result it is easily blocked. There needs to be a way to force developers to move to newer APIs. Bergmann mentioned the [desire to remove high memory](<https://lwn.net/Articles/813201/>), but 32-bit Arm still needs it, so it cannot be removed from anywhere, imposing significant costs on the kernel as a whole. Chinner complained that increasing amounts of work are being placed on maintainers to keep old code working; maintainers are at the limit of what they can do now, and this path is unsustainable, he said. 

Torvalds repeated that it is possible to deprecate old code when there is a good reason to do so. When Gleixner again said that he cannot get architecture maintainers to move to new APIs, Torvalds added that kernel developers often add new APIs alongside old ones to avoid having to fix everything at the outset. Perhaps, he said, developers should try to avoid that approach and, instead, just fix everything right away. Gleixner said that, for some changes, every subarchitecture must be fixed manually, which is a huge job. 

Christian Brauner said that he had similar problems with the adoption of [the new mount API](<https://lwn.net/Articles/759499/>); it took years to convert a majority of the important filesystems, and he had to do a lot of the work himself. A lot of his patches were rejected, creating a frustrating situation. Hellwig added that, during this conversion, two more filesystems using the old API were merged. Torvalds suggested that, in many of these cases, adding a warning might be all that is needed to put pressure on maintainers to move forward. 

The above discussion was supposed to fit into a 30-minute discussion slot; readers who have gotten this far will be unsurprised to learn that it ran significantly over. At this point, the developers were in need of a break, so this topic was put aside so that the rest of the agenda could be addressed.
