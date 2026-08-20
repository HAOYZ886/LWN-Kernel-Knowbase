---
title: Merging the kvm tool
url: https://lwn.net/Articles/456132/
date: "August 23, 2011"
category: "KVM; Virtualization-KVM"
author: "By Jonathan Corbet August 23, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 23, 2011

The "native Linux KVM tool" (which we'll call "NLKT") is a hardware emulation system designed to support virtualized guests running under the KVM hypervisor. It offers a number of nice features, but an attempt to get this code merged into the 3.1 kernel was deferred by Linus, who did not want to deal with another controversial development at that time. This tool's developers have let it be known that it will be back for the 3.2 merge window; controversy is sure to follow. The core question raised by this project is: what code is appropriate for the kernel tree, and which projects should live in their own repositories elsewhere? 

NLKT was started in response to [unhappiness about QEMU](<https://lwn.net/Articles/379869/>), the state of its code, and the pace of its development. It was designed with simplicity in mind; NLKT is meant to be able to boot a basic Linux kernel without the need for a BIOS image or much in the way of legacy hardware emulation. Despite its simplicity, NLKT offers "just works" networking, SMP support, basic graphics support, copy-on-write block device access, host filesystem access with 9P or overlayfs, and more. It has developed quickly and is, arguably, the easiest way to get a Linux kernel running on a virtualized system. 

Everybody seems to think that NLKT is a useful tool; nobody objects to its existence. The controversy comes for other reasons, one of which is the name: the tool simply calls itself "kvm." The potential for confusion with the kernel's KVM subsystem is clear - that is why this article made up a different acronym to refer to the tool. "KVM" is already seen as an unfortunate name - searches for the term bring in a lot of information about keyboard-video-mouse switches - so adding more ambiguity seems like a bad move. It is also seemingly viewed by some as a move to be the "official" hardware emulator for KVM. The NLKT developers have, thus far, resisted a name change, though. 

The bigger fight is over whether NLKT belongs in the kernel at all. It is not kernel code; it is a program that runs in user space. The question of whether such code should be in the kernel's repository is certainly the one that will decide whether it is merged for 3.2 or not. 

NLKT would not be the first user-space tool to go into the mainline kernel; several others can be found in the `tools/` directory. Many of them are testing tools used by kernel developers, but not all. The "cpupower" tool was merged for 3.1; it allows an administrator to tweak various CPU power management features. The most actively developed tool in that directory, though, is perf, which has grown by leaps and bounds since being merged into the mainline. The developers working on perf have been very outspoken in their belief that putting the tool into the mainline kernel repository has helped it to advance quickly. 

Proponents say that, like perf, NLKT is closely tied to the kernel and heavily used by kernel developers; like perf, it would benefit from being put into the same code repository. KVM, they say, is also under heavy development; having NLKT and KVM in the same tree would help both to improve more quickly. It would bring more review of any future KVM ABI changes, since a user of that ABI would be merged into the kernel as well. Keeping the hardware emulation code near the drivers that code has to work with is said to be beneficial to both sides. All told, they say, perf would not have been nearly as successful outside of the mainline tree as it has been internally; merging NLKT can be expected to encourage the same sort of success. 

That success seems to be one of the things that opponents are worried about; some have [worried](<https://lwn.net/Articles/456135/>) that the main purpose is to increase the project's visibility so that it succeeds at the expense of competing projects. The ABI development benefits are challenged; any changes would clearly still have to work with tools like QEMU regardless of whether NLKT is in the kernel, so QEMU developers would have to remain in the loop. It is even better, some say, to separate the implementation of an ABI from its users; that forces the implementers to put more effort into documenting how the ABI should be used. 

There is also concern that, once we start seeing more user-space tools going into the kernel tree, there will be an unstoppable flood of them. Where does it stop, they ask - should we pull in the C library, the GNU tools, or, maybe, LibreOffice? Linux is not BSD, they say; trying to put everything into a single repository is not the right direction to take. The answer to that complaint is that there is no interest in merging arbitrary tools; only those that are truly tied to the kernel would qualify. By this reasoning, NLKT is an easy decision. A C library is something that could be considered; perhaps even graphics if the relevant developers wanted to do that. But office suites are not really eligible; there are limits to what should go into the mainline. 

That was where the discussion stood at the beginning of the 3.1 merge window; Linus [decided](<https://lwn.net/Articles/453957/>) not to pull NLKT at that time. Instead, he clearly wanted the discussion to continue; he told the NLKT developers that they would have to convince him in the 3.2 merge window instead. It looks like that process is about to begin; the NLKT repository is about to be added to linux-next in anticipation of a pull request once the merge window opens. This time, with luck, we'll have a resolution of the issue that gives some guidance for those who would merge other user-space tools in the future.
