---
title: Device tree troubles
url: https://lwn.net/Articles/560523/
date: "July 24, 2013"
category: Device tree
author: "By Jonathan Corbet July 24, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 24, 2013

Kernel developers working on the x86 architecture are spoiled; they develop for hardware that, for the most part, identifies itself when asked, with the result that it is usually easy to figure out how a specific machine is put together. Other architectures — most notably ARM — are rather messier in this regard, requiring the kernel to learn about the configuration of the hardware from somewhere other than the hardware itself. Once upon a time, hard-coded "board files" were used to build ARM-system-specific kernels; more recently, the [device tree](<http://devicetree.org/Device_Tree_Usage>) mechanism has emerged as the preferred way to describe a system to the kernel. A device tree file provides the enumeration information that the hardware itself does not, allowing the kernel to understand the configuration of the system it is running on. The device tree story is one of success, but, like many such stories, success is bringing on some growing pains. 

A device tree "binding" is the specification of how a specific piece of hardware can be described in the device tree data structure. Most drivers meant to run on platforms where device trees are used include a documentation file describing that driver's bindings; see [Documentation/devicetree/bindings/net/can/cc770.txt](<https://lwn.net/Articles/560533/>) as a randomly chosen example. The kernel contains nearly 800 such files, plus a hundreds more "`.dts`" files describing complete system-on-chips and boards, and the number is growing rapidly. 

Maintenance of those files is proving to be difficult for a number of reasons, but the core of the problem can be understood by realizing that a device tree binding is a sort of API that has been exposed by the kernel to the world. If a driver's bindings change in an incompatible way, newer kernels may fail to boot on systems with older device trees. Since the device tree is often buried in the system's firmware somewhere, this kind of problem can be hard to fix. But, even when the fix is easy, the kernel's normal API rules should apply; newer kernels should not break on systems where older kernels work. 

The clear implication is that new device tree bindings need to be reviewed with care. Any new bindings should adhere to existing conventions, they should describe the hardware completely, and they should be supportable into the future. And this is where the difficulties show up, in a couple of different forms: (1) most subsystem maintainers are not device tree experts, and thus are not well equipped to review new bindings, and (2) the maintainers who _are_ experts in this area are overworked and having a hard time keeping up. 

The first problem was the subject of [a request for a Kernel Summit discussion](<https://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-July/000110.html>) with the goal of educating subsystem maintainers on the best practices for device tree bindings. One might think that a well-written document would suffice for this purpose, but, unfortunately, these best practices still seem to be in the "I know it when I see it" phase of codification; as Mark Brown [put it](<https://lwn.net/Articles/560544/>): 

At the minute it's about at the level of saying that if you're not sure or don't know you should get the devicetree-discuss mailing list to review it. Ideally someone would write that document, though I wouldn't hold my breath and there is a bunch of convention involved. 

Said mailing list tends to be overflowing with driver postings, though, making it less useful than one might like. Meanwhile, the best guidance, perhaps, came from [David Woodhouse](<https://lwn.net/Articles/560546/>): 

The biggest thing is that it should describe the *hardware*, in a fashion which is completely OS-agnostic. The same device-tree binding should work for Solaris, *BSD, Windows, eCos, and everything else. 

That is, evidently, not always the case, currently; some device tree bindings can be strongly tied to specific kernel versions. Such bindings will be a maintenance problem in the long term. 

Keeping poorly-designed bindings out of the mainline is the responsibility of the device tree maintainers, but, as Grant Likely (formerly one of those maintainers) [put it](<https://lwn.net/Articles/560547/>), this maintainership ""simply isn't working right now."" Grant, along with Rob Herring, is unable to keep up with the stream of new bindings (over 100 of which appeared in 3.11), so a lot of substandard bindings are finding their way in. To address this problem, Grant has announced a "refactoring" of how device tree maintainership works. 

The first part of that refactoring is Grant's own resignation, with lack of time given as the reason. In his place, four new maintainers (Pawel Moll, Mark Rutland, Stephen Warren and Ian Campbell) have been named as being willing to join Rob and take responsibility for device tree bindings; others with an interest in this area are encouraged to join this group. 

The next step will be for this group to figure out how device tree maintenance will actually work; as Grant noted, ""There is not yet any process for binding maintainership."" For example, should there be a separate repository for device tree bindings (which would make review easier), or should they continue to be merged through the relevant subsystem trees (keeping the code and the bindings together)? It will take some time, and possibly a Kernel Summit discussion, to figure out a proper mechanism for the sustainable maintenance of device tree bindings. 

Some other changes are in the works. The kernel currently contains hundreds of `.dts` files providing complete device trees for specific systems; there are also many `.dtsi` files describing subsystems that can be included into a complete device tree. In the short term, there are plans to design a schema that can be used to formally describe device tree bindings; the device tree compiler utility (`dtc`) will then be able to verify that a given device tree file adheres to the schema. In the longer term, those device tree files are likely to move out of the kernel entirely (though the binding documentation for specific devices will almost certainly remain). 

All told, the difficulties with device trees do not appear to be anything other than normal growing pains. A facility that was once only used for a handful of PowerPC machines (in the Linux context, anyway) is rapidly expanding to cover a sprawling architecture that is in wide use. Some challenges are to be expected in a situation like that. With luck and a fair amount of work, a better set of processes and guidelines for device tree bindings will result from the discussion — eventually.
