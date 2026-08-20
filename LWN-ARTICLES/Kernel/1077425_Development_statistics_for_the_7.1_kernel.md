---
title: Development statistics for the 7.1 kernel
url: https://lwn.net/Articles/1077425/
date: "June 15, 2026"
category: "Releases-7.1"
author: "By Jonathan Corbet June 15, 2026"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 15, 2026

Linus Torvalds [released the 7.1 kernel](<https://lwn.net/ml/all/CAHk-=wi4BF4bMhZNZ1tqs+FFV4OuZRe3ZqdWB+LxRLmRweUzQw@mail.gmail.com/>) as expected on June 14\. This development cycle brought in a lot of new features — and a lot of new developers as well. The time has come for our traditional look at where the changes in 7.1 came from, with a digression into how our community may be changing in general. 

This release saw the merging of 15,849 non-merge changesets from 2,479 developers. That makes 7.1 one of the busiest development cycles in the kernel's history; only four other releases brought in more commits. The [6.7 release](<https://lwn.net/Articles/956765/>) remains the busiest ever, with 17,284 commits; the size of that release was driven by the ill-fated addition of the bcachefs filesystem and all of its development history. The [5.8](<https://lwn.net/Articles/827735/>), [5.10](<https://lwn.net/Articles/839772/>), and [5.13](<https://lwn.net/Articles/860989/>) also brought in more commits than 7.1, though by much smaller margins. 

The number of developers working on 7.1 _does_ set a new record, beating the short-lived record of 2,362 set by 7.0. The trend here merits some attention, but we'll start with the usual numbers. The most prolific developers working on 7.1 were: 

> Most active 7.1 developers  
> ---  
> | By changesets  
> ---  
> Johan Hovold | 181| 1.1%  
> Thomas Weißschuh | 175| 1.1%  
> Eric Biggers | 168| 1.1%  
> Stefan Metzmacher | 155| 1.0%  
> Krzysztof Kozlowski | 148| 0.9%  
> Rafael J. Wysocki | 147| 0.9%  
> Russell King | 130| 0.8%  
> Tejun Heo | 125| 0.8%  
> Christoph Hellwig | 122| 0.8%  
> Eric Dumazet | 120| 0.8%  
> Jakub Kicinski | 119| 0.8%  
> Sean Christopherson | 113| 0.7%  
> Thorsten Blum | 101| 0.6%  
> Thomas Zimmermann | 86| 0.5%  
> Dmitry Torokhov | 85| 0.5%  
> Andy Shevchenko | 84| 0.5%  
> Mauro Carvalho Chehab | 84| 0.5%  
> Chuck Lever | 79| 0.5%  
> Bartosz Golaszewski | 77| 0.5%  
> Lorenzo Stoakes | 77| 0.5%  
> | By changed lines  
> ---  
> Jakub Kicinski | 126367| 12.7%  
> Roman Li | 105777| 10.7%  
> Namjae Jeon | 54445| 5.5%  
> Andrew Lunn | 16180| 1.6%  
> Eric Biggers | 14471| 1.5%  
> Stefan Metzmacher | 14403| 1.5%  
> Taniya Das | 11956| 1.2%  
> Alexei Starovoitov | 11410| 1.2%  
> Andy Shevchenko | 7673| 0.8%  
> Mauro Carvalho Chehab | 7637| 0.8%  
> Christian Brauner | 7483| 0.8%  
> Pankaj Patil | 7181| 0.7%  
> Christoph Hellwig | 6368| 0.6%  
> Krzysztof Kozlowski | 5993| 0.6%  
> Besar Wicaksono | 5873| 0.6%  
> Dmitry Baryshkov | 5787| 0.6%  
> Tejun Heo | 5698| 0.6%  
> Derek J. Clark | 5288| 0.5%  
> Vincent Donnefort | 5033| 0.5%  
> Ratheesh Kannoth | 4992| 0.5%  
  
**About the [KSDB] links**

As an experiment, in this article, links marked [KSDB] point into the subscriber-only [LWN Kernel Source Database](</ksdb/>), where more information can be found. 

Johan Hovold [[KSDB](</ksdb/releases/v7.1/commits?dev=3>)] was the biggest contributor of changesets in this development cycle, with a lot of cleanup work in the SPI subsystem and beyond. Thomas Weißschuh's [[KSDB](</ksdb/releases/v7.1/commits?dev=312>)] work covered many parts of the kernel, including the build system, nolibc, SPARC architecture code, and more. Eric Biggers [[KSDB](</ksdb/releases/v7.1/commits?dev=239>)] has been busily refactoring the kernel's cryptographic code. Stefan Metzmacher [[KSDB](</ksdb/releases/v7.1/commits?dev=5407>)] contributed improvements to the SMB filesystem implementation, and Krzysztof Kozlowski [[KSDB](</ksdb/releases/v7.1/commits?dev=549>)] continues to work throughout the system-on-chip and devicetree subsystems. 

In the lines-changed column, Jakub Kicinski [[KSDB](</ksdb/releases/v7.1/commits?dev=170>)] removed a large chunk of old and unmaintained networking code, including the ISDN, Bluetooth CMTP, and ATM subsystems. Roman Li [[KSDB](</ksdb/releases/v7.1/commits?dev=389>)] added yet another big set of amdgpu header files. Namjae Jeon [[KSDB](</ksdb/releases/v7.1/commits?dev=355>)] has [brought back the older NTFS filesystem implementation](<https://lwn.net/Articles/1055062/#ntfs>) and added a lot of new features to it. Andrew Lunn [[KSDB](</ksdb/releases/v7.1/commits?dev=827>)] removed a set of unmaintained networking drivers. 

Nearly 8% of the commits in 7.1 carried Tested-by tags, while almost 51% had Reviewed-by tags. Both of those numbers are relatively low compared to recent releases. The top testers and reviewers this time were: 

> Test and review credits in 7.1  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 89| 5.8%  
> Fuad Tabba | 61| 3.9%  
> Gavin Shan | 40| 2.6%  
> Shaopeng Tan | 40| 2.6%  
> Mohd Ayaan Anwar | 40| 2.6%  
> Jesse Chick | 40| 2.6%  
> Mostafa Saleh | 36| 2.3%  
> Punit Agrawal | 34| 2.2%  
> Jon Hunter | 33| 2.1%  
> Zeng Heng | 31| 2.0%  
> Lad Prabhakar | 29| 1.9%  
> Peter Newman | 28| 1.8%  
> Eric Biggers | 27| 1.7%  
> Venkat Rao Bagalkote | 22| 1.4%  
> Randy Dunlap | 21| 1.4%  
> | Reviewed-by  
> ---  
> Konrad Dybcio | 334| 3.1%  
> Dmitry Baryshkov | 300| 2.8%  
> Andy Shevchenko | 197| 1.9%  
> Krzysztof Kozlowski | 181| 1.7%  
> Christian König | 170| 1.6%  
> Ilpo Järvinen | 161| 1.5%  
> Simon Horman | 140| 1.3%  
> Frank Li | 134| 1.3%  
> Geert Uytterhoeven | 122| 1.1%  
> Christoph Hellwig | 119| 1.1%  
> Jonathan Cameron | 118| 1.1%  
> Gary Guo | 114| 1.1%  
> Andrea Righi | 103| 1.0%  
> Rob Herring | 100| 0.9%  
> Lorenzo Stoakes | 94| 0.9%  
  
Daniel Wheeler [[KSDB](</ksdb/releases/v7.1/taglist?tag=tested-by&dev=612>)] maintains his perpetual position as the kernel's top credited tester. On the review side, Konrad Dybcio [[KSDB](</ksdb/releases/v7.1/taglist?tag=reviewed-by&dev=236>)] added his tag to 334 changes while Dmitry Baryshkov [[KSDB](</ksdb/releases/v7.1/taglist?tag=reviewed-by&dev=8>)] tagged 300; both performed reviews almost exclusively for Qualcomm drivers and devicetree files, mostly written by their Qualcomm colleagues. 

Development for the 7.1 release was supported by 230 employers that we were able to identify; the most active of those were: 

> Most active 7.1 employers  
> ---  
> | By changesets  
> ---  
> (Unknown)| 2160| 13.6%  
> Intel| 1435| 9.1%  
> Google| 1227| 7.7%  
> AMD| 828| 5.2%  
> Qualcomm| 787| 5.0%  
> Red Hat| 745| 4.7%  
> (None)| 694| 4.4%  
> NVIDIA| 595| 3.8%  
> Meta| 535| 3.4%  
> (Consultant)| 500| 3.2%  
> SUSE| 459| 2.9%  
> Oracle| 390| 2.5%  
> Arm| 334| 2.1%  
> Renesas Electronics| 289| 1.8%  
> NXP Semiconductors| 254| 1.6%  
> IBM| 215| 1.4%  
> Huawei Technologies| 210| 1.3%  
> Kylin| 181| 1.1%  
> Linutronix| 171| 1.1%  
> SerNet| 155| 1.0%  
> | By lines changed  
> ---  
> Meta| 156583| 15.8%  
> AMD| 136736| 13.8%  
> (Unknown)| 76738| 7.7%  
> Qualcomm| 75847| 7.6%  
> Samsung| 55391| 5.6%  
> Google| 54537| 5.5%  
> Intel| 48917| 4.9%  
> NVIDIA| 37374| 3.8%  
> Red Hat| 35706| 3.6%  
> (None)| 35178| 3.5%  
> Oracle| 15837| 1.6%  
> SerNet| 14403| 1.5%  
> Arm| 13030| 1.3%  
> SUSE| 12208| 1.2%  
> NXP Semiconductors| 11777| 1.2%  
> Huawei Technologies| 11580| 1.2%  
> Realtek| 10314| 1.0%  
> (Consultant)| 9984| 1.0%  
> Marvell| 8282| 0.8%  
> Amutable| 7483| 0.8%  
  
As usual, there is little change here from previous development cycles. 

#### The developers keep coming

There is one thing that is clearly changing, though. The [7.0 development-statistics article](<https://lwn.net/Articles/1066723/>) noted that the number of first-time kernel contributors has been growing. The 7.1 cycle continued that trend with 530 new contributors [[KSDB](</ksdb/releases/v7.1/first_timers>)], far more than the previous record of 489 set by 7.0. The numbers now look like this: 

> ![\[First-time contributors bar
chart\]](https://static.lwn.net/images/2026/first-timers-7.1.svg)

This trend, which shows no signs of stopping, is almost certainly driven by the increasing availability of LLM-based development tools; it has made itself felt in a number of ways. There are 299 commits in 7.1 that include an Assisted-by tag indicating the use of such a tool. The number of actual commits created with LLM involvement must be significantly higher, though; a number of developers are clearly not complying with the kernel's [rules](<https://docs.kernel.org/process/coding-assistants.html>) for disclosing that use. 

These new developers are showing up in surprising places. The [serial-line IP (SLIP)](<https://en.wikipedia.org/wiki/Serial_Line_Internet_Protocol>) protocol implementation, for example, has seen almost no attention outside of maintenance changes for years, but it is now seeing fixes like [this one](<https://git.kernel.org/linus/4c1367a2d7aa>) from Weiming Shi [[KSDB](</ksdb/developers/show?dev=38676>)], a developer who first showed up in 6.19. The long-unloved floppy driver received [this fix](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e784f2ea0b4f>) (since reverted) from 6.17 first-timer Guangshuo Li [[KSDB](</ksdb/developers/show?dev=37871>)]. The [OMFS](<https://docs.kernel.org/filesystems/omfs.html>) filesystem received [its first non-mechanical change](<https://git.kernel.org/linus/0621c385fda1>) in many years from first-timer HyungJung Joo [[KSDB]](</ksdb/developers/show?dev=39536>). 

The most prolific 7.1 first-timer, Michael Bommarito [[KSDB](</ksdb/developers/show?dev=39421>)] with 60 commits, contributed fixes to the SMB filesystem, the SCTP network protocol, the Bluetooth subsystem, io_uring, the SCSI subsystem, the amdgpu driver, the RDMA RoCEv2 implementation, and beyond. Almost all of those patches carried Assisted-by tags. There are few kernel developers who could make substantial changes across that much of the kernel; the ability of a previously unseen developer to do that is something new. Whether any developer, new or old, can fully _understand_ substantive patches to that range of kernel subsystems is a different question. 

Then, there is the seemingly infinite stream of typo fixes that has turned into an outright flood in recent times. As the documentation maintainer, I welcome these changes as a way for a new developer to become familiar with the development process, but I also try to encourage developers to move on to more substantial changes — advice that is taken less often than I would like. 

Of course, access to an LLM is not the only reason a developer might enter our community. A minimum of 132 of the first-time developers seen in this development cycle can already be associated with an employer. Qualcomm leads the pack with 15 first-time developers; AMD employed 14, and Google 12\. In total, 51 companies employed first-time 7.1 developers. 

One of the many motivations behind bringing Rust into the kernel was the hope of attracting younger developers into the community. A total of five of the new developers (just under 1%) touched Rust files (those whose names end in "`.rs`"). Three of those developers contributed single, small patches, one added a number of tracepoints to the binder driver, and one (Eliot Courtney [[KSDB](</ksdb/developers/show?dev=39656>)]) contributed 23 commits, 19 of which were to the in-progress "nova" driver for NVIDIA GPUs. So Rust may be bringing in developers, but not yet in huge numbers yet. 

Overall, the areas most frequently touched by new developers were: 

> Subsystem| # developers  
> ---|---  
> `Documentation`| 78  
> `net`| 66  
> `other drivers`| 52  
> `drivers/net`| 49  
> `drivers/staging`| 47  
> `sound`| 46  
> `include`| 36  
> `drivers/gpu/drm`| 35  
> `other fs`| 33  
> `tools`| 32  
> `arch/arm64`| 26  
> `kernel`| 25  
> `fs/smb`| 19  
> `drivers/usb`| 17  
> `drivers/iio`| 15  
> `drivers/hwmon`| 13  
> `drivers/bluetooth`| 10  
> `MAINTAINERS`| 10  
> `drivers/platform`| 10  
> `drivers/i2c`| 9  
> `mm`| 8  
  
There are some interesting patterns here. The crowd of new developers working on the documentation is nice on its face .... but see "typo fixes", above. The number of first-timers changing the `MAINTAINERS` file could be of concern; there has been an episode or two recently where an unknown developer tried to claim maintainership of a subsystem, a prospect that is worrisome for obvious reasons. In this case, roughly half of the `MAINTAINERS` changes accompanied new drivers submitted by the new developer in question. There _are_ a couple of previously unseen developers who showed up and claimed the maintainership of an existing subsystem, but they used email addresses from the company involved and, at a first glance, look legitimate. Meanwhile, the staging tree was meant to be a way for new developers to enter the community; it appears to still be serving that purpose. 

One number that stands out is the number of new developers contributing to the SMB filesystem. These developers are not fixing typos; instead, they are addressing what appear to be serious, perhaps security-relevant bugs. Whether these developers are running their own tools to find bugs, or whether instead there is some sort of organized effort to direct new developers at those bugs is unclear. What does seem clear is that anybody who is using SMB in a setting where security matters may want to apply extra diligence to following kernel updates for a while. 

As the numbers in the first part of this article show, the kernel's development process appears to be rolling along at its usual fast pace — or even a bit faster. But things are changing. New development tools have facilitated the entry of a large crowd of new developers into the community. If enough of them stay around, it should not take long to change the makeup of the kernel community — and how that community works — substantially. We live in interesting times.
