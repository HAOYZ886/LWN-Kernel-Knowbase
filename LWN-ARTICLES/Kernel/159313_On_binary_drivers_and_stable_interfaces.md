---
title: On binary drivers and stable interfaces
url: https://lwn.net/Articles/159313/
date: "November 9, 2005"
category: Copyright issues
author: 
---

There has recently been a surge of discussion, once again, on whether the Linux kernel should support closed-source drivers. The debate was driven, perhaps, by the [suspicion](<http://www.kroah.com/log/2005/11/03/>) (later [put to rest](<http://www.kroah.com/log/2005/11/07/#osdl_gkai2>)) that OSDL was supporting the creation of a stable binary driver ABI for Linux. So perhaps the time has come to review the reasons why the kernel developers are opposed to closed-source drivers. Our apologies to all of you who have seen this before. 

Support for binary-only drivers seems, on the surface, like it could be a good idea. Companies could provide Linux drivers for their hardware without exposing their "valuable intellectual property" to the world. Users would have a higher degree of assurance that their hardware would simply work. All of the current hardware hassles would go away, and everybody would be happy. What could be wrong with that? 

One obvious problem is that, with a proprietary driver, a Linux system loses one of its best characteristics: independence from vendors. A user of a proprietary driver depends on the vendor for fixes and updates, but the vendor is under no obligation to provide them. Computing hardware has a notoriously short product life; if the vendor drops driver support when a product hits the end of its life, there is little that a user can do. If the vendor goes out of business, there will be no further support for the driver. If the vendor decides to start charging for driver updates, the user has little option but to pull out the wallet. If the driver has a bug which affects the stability of the system, only the vendor can fix it. 

And history shows that proprietary drivers tend to have plenty of bugs. They are often written by developers with little time and even less expertise with the Linux kernel. The code does not go through any sort of peer review, so obvious problems will persist into the final product. And, since only the vendor can fix the driver, bugs can last for a long time. 

Binary drivers are brittle. The kernel API can and does change; that aspect of the kernel is not going away. Freezing an API would limit the developers' ability to fix poor interfaces, improve how the kernel works, and remove cruft. So binary drivers will always be likely to break between kernel releases, and users will have to wait for the vendor to get around to catching up with the current API. 

Linux kernel developers will not help users who have proprietary drivers loaded into their systems. That is not because the developers want to be petty and vengeful (well, perhaps one or two of them do); it is simply that the developers have no way to track down problems when closed-source code is running. 

Even if a vendor offers top-quality drivers and support, it is unlikely that said vendor supports all of the architectures that run Linux. Freedom to run on something other than i386 is one of the great advantages of Linux; proprietary code takes that freedom away. 

Finally, proprietary drivers may constitute copyright infringement. Certainly some developers feel that kernel modules are derived products of the kernel itself, and thus required to carry the kernel's (GPL) license. Whether the module interface constitutes a boundary which the GPL cannot cross can only, in the end, be determined by the courts. Until then, every proprietary driver carries with it a degree of legal uncertainty. 

None of this is new; here's what [Linus Torvalds said](<http://lwn.net/1999/0211/a/lt-binary.html>) back in 1999: 

Basically, I want people to know that when they use binary-only modules, it's THEIR problem. I want people to know that in their bones, and I want it shouted out from the rooftops. I want people to wake up in a cold sweat every once in a while if they use binary-only modules. 

The alternative to cold sweats is to stick with hardware which comes with free drivers. In most areas, finding such hardware is not a challenge. In the cases where it can be a problem (video adapters, some wireless network cards), the solution is not to weigh down the kernel with some sort of set-in-stone ABI. As Linux continues to grow in popularity - and proprietary drivers get harder to write and maintain - recalcitrant vendors should eventually come around. That's exactly what has tended to happen thus far.
