---
title: "ELC: Status of embedded Linux"
url: https://lwn.net/Articles/383363/
date: "April 14, 2010"
category: Embedded systems
author: "By Jake Edge April 14, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
April 14, 2010

[Embedded Linux Conference](<http://www.embeddedlinuxconference.com/elc_2010/>) (ELC) organizer Tim Bird surveyed the embedded Linux landscape in a talk he gave at the conference. He looked at new and proposed kernel features that embedded developers might be interested in as well as issuing a "call to arms" to those developers to get more involved with the rest of the community. This talk is a regular feature at ELC to help the embedded community stay on top of the ""ripping"" speed of kernel development. 

There have been four new kernels since last year's conference, and Bird listed the interesting features for the embedded space in each of 2.6.30-33, as well as noting that LogFS had finally made it into the kernel in 2.6.34, something that he was concerned might not ever happen. The speed of kernel development is amazing, he said, and the great thing about it is that ""even while I am sleeping in my bed, people are pounding away on it"". 

He pointed to a few ""patches to watch"" that may be coming in new kernels, specifically the kbuild `CROSS_COMPILE` option, which will make it easier to build for multiple architectures. He also noted Arnd Bergmann's [asm-generic](<http://lwn.net/Articles/357803/>) patches that are geared towards making it easier to add new architectures to the kernel--without propagating the bugs and quirks from existing ones. 

#### Boot speed

Bird then looked at different "technology areas" to point out interesting features or work going on in those areas. Boot time is a ""hot topic"" right now; it would have been in the past if the embedded community was more involved in mainline kernel development. The Moblin [five second boot](<http://lwn.net/Articles/299483/>) effort really kickstarted that work. He noted that he has a Sony (his employer) video camera that boots Linux in 1.5 seconds; ""I'm very proud of that"", he said. 

Several new kernel features are available to help reduce boot time, including [asynchronous function calls](<http://lwn.net/Articles/314808/>), which allow some parts of device initialization to run in parallel. There is also `scripts/bootgraph.pl` to help visualize where boot time is being spent. 

[Devtmpfs](<http://lwn.net/Articles/331818/>) was also noted as a way to decrease boot times, with some seeing a 0.6 second reduction on desktops. Bird said that there needs to be some testing done on the embedded side to see how much it can help there. He also listed two patches that speed up symbol resolution for module loading by getting rid of the current linear search. One switches to a binary search and the other uses a hash table. For Bird's use cases, he always statically links in drivers, but has heard that more embedded developers are going the loadable module route. 

Greg Kroah-Hartman piped up that he needed one of those two patches for MeeGo, but that the submitters had disappeared. There was general agreement that contacting them and getting something upstream would be good. 

#### Filesystems

Several different filesystems for embedded use cases were listed by Bird. Squashfs has been out of the mainline for years, but was merged in 2.6.29, and has since been improved by others in a ""classic case"" demonstrating the advantages of mainline code. Ubifs is also in the mainline and folks at Toshiba have been characterizing its performance, which they reported on at the CE Linux Forum (CELF) Japan Jamboree. It has ""really slow mount times"" in some cases, which CELF would like to fund someone to fix. 

LogFS is ""way better optimized"" for certain flash devices and has fast mount times, he said. He noted that AXFS, the advanced execute-in-place (XIP) filesystem, had kind of disappeared, so it didn't appear to be on track for mainlining. He has been playing with AXFS at Sony to try to further decrease boot time. 

Bird also noted that the [VFAT patent avoidance patches](<http://lwn.net/Articles/338981/>) had not made it into the mainline. It would be useful for some embedded devices, he said. Most embedded developers work around the patent by disabling VFAT and using 8.3 filenames, which is somewhat unfortunate. Another thing he is keeping an eye on is VFS-based union mounts, which would allow embedded developers to stop creating ""filesystems with weird links"" between them as is currently common. 

#### Power management and realtime

The runtime power management code has been merged, which will allow suspending and resuming individual system components to reduce power consumption. There is ongoing work on asynchronous suspend/resume, which Bird said he didn't know very much about, but it's ""gotta be really cool"". An audience member helped out by saying that it is in some ways like the asynchronous initialization code (for faster boot), but ""in the other direction"". 

The `RT_PREEMPT` patchset ""continues its slow march into the kernel"", with [threaded interrupt handlers](<http://lwn.net/Articles/302043/>) being merged in 2.6.30 and preparatory work for the future [sleeping spinlocks](<http://lwn.net/Articles/357800/>) merge that went into 2.6.33. There are still some big kernel lock issues (BKL) to be resolved and CELF may fund some work in that area. 

#### Kernel size

The slide for kernel size and memory use had a picture of a ""hybrid Winnebago"", which is the image Bird has of the kernel today. It just keeps growing in size. To help embedded developers make better use of limited memory, there is the [smem tool](<http://lwn.net/Articles/329458/>) that was funded by CELF. He has used it in a few projects this year and it ""has been very helpful"". 

Various compression methods have been added to compress the kernel image in different ways. LZMA can be up to 30% better than gzip, and LZO is not as good at compression, but is much faster. There are tradeoffs dependent on processor speed and I/O bandwidth that make it more difficult to pick the right compression method, as Dirk Hohndel pointed out. 

The ramzswap device (also known as [compcache](<http://lwn.net/Articles/334649/>)) allows in-memory compressed swap. It is ""really cool"" but the maintainer only was able to benchmark on desktop systems. It would be good if someone could do some benchmarking on embedded systems, Bird said. 

#### Tracing and security

Ftrace now has support for dynamic probes that came in 2.6.33, and the perf tool can place and use those probes as well. There is tracing of kernel variable access and modification available now. The perf "diff" mode can show the performance differences between two runs, and also came in 2.6.33. 

The TOMOYO merge in 2.6.30 was ""a big deal"" because it finally was able to get path-based security into the kernel. NTT Data is now adding TOMOYO rules to Android. Bird is in favor of a diversity of choices for security as it gives people a chance to demonstrate which is the best solution for various use cases. As part of that, CELF funded a [study [PDF]](<http://www.embeddedalley.com/pdfs/Smack_for_DigitalTV.pdf>) for applying the Smack security module in a television use case and found that the overhead was higher than they expected. 

#### CELF contract work

CELF has funded various projects over the last year including smem, out-of-memory notifications in cgroups, SquashFS, the Smack analysis, device trees for ARM, and the `-ffunction-sections` work to put each kernel function in its own section to assist with dead code removal. Going forward, CELF has an [open project proposal](<http://elinux.org/CELF_Open_Project_Proposal_2010>) plan that will start funding new projects in the next few weeks. It is also sponsoring Matt Mackall to be one of the two Linux kernel embedded maintainers (David Woodhouse is the other). 

#### A call to arms

Bird ended his talk with a list of things that embedded developers can do to work better with the community. At the top of that list was ""work at top of tree"". He realized that when he gives these talks, he is generally talking to people who aren't using the kernels he talks about because embedded folks tend to pick a kernel and stick with it. ""It's difficult"" to work with the most recent kernel, but it's worth it. ""Version gap is the single biggest problem"" in embedded Linux. He suggested that embedded developers beat up on their board vendor to get board support packages using the latest kernels and to do their testing on boards that are already supported in the mainline. 

The other suggestion he had was not to ""wait for others to test new features"", and instead to do the testing themselves. He listed a number of things that need testing in the mainline: LogFS, Ubifs mount times, ramzswap, runtime power management, and so on. ""Post the results to the [elinux.org wiki](<http://elinux.org/Main_Page>)"" or come to the next conference (October 27-28 in Cambridge, UK) and tell him about it.
