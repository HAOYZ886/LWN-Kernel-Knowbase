---
title: The LPC Android microconference
url: https://lwn.net/Articles/570406/
date: "October 17, 2013"
category: Android; kdbus
author: "October 17, 2013 This article was contributed by John Stultz Linux
Plumbers Conference"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

October 17, 2013

This article was contributed by John Stultz

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2013-Linux_Plumbers_Conference>)

A number of upstream community developers and Google Android developers got together at the 2013 Linux Plumbers Conference discuss some of the non-graphics related out-of-tree or still-in-staging portions of the Android patch set. This discussion followed the [Android graphics microconference](<https://lwn.net/Articles/569704/>) held earlier in the day. This article contains a summary of the major issues discussed at this gathering. 

#### Unified IPC with binder and kdbus

Earlier in the year, Greg Kroah-Hartman from the Linux Foundation had expressed his goal of getting the user-space binder libraries to run over the still-in-development [kdbus interprocess communication mechanism](<https://lwn.net/Articles/551969/>). The session started with a question to Greg: "how is that going?". Bashfully rubbing his head and getting playfully heckled by Kay Sievers (another of the kdbus developers) from Red Hat, Greg suggested that we maybe could check back next year. Greg mentioned that he still has it as a goal, but Kay's concerns about the viability of the concept have made him somewhat less confident. At this point Kay stepped in to describe some of the semantic differences between the mechanisms, such as how everything in kdbus is asynchronous, making it more like a network protocol, while binder calls block to completion in a way that's more akin to system calls. These differences in behavior make it very difficult to support both modes of operation elegantly and efficiently in one IPC mechanism. 

When questioned whether this meant that binder should be merged upstream alongside kdbus, Greg was a bit hesitant given the reported security concerns with using binder in a non-Android environment. Colin Cross, one of the Android developers in attendance, noted that the security issue is quite easy to avoid, and wouldn't really be a problem, but from his perspective, he doesn't see an immediate need to get binder out of staging and officially merged upstream. Greg agreed, and also clarified there is no rush, as he is fine with binder staying in staging for however long is necessary. 

The Android developers have often been dragged through the mud by the community for implementing their own solutions, focused on solving their own problems, rather than working with the community to fix the existing infrastructure to satisfy their needs. In that context, I somewhat jokingly prodded Greg to explain why it is acceptable to develop kdbus instead of fixing or expanding binder's features to support D-Bus. He acknowledged the contradiction but repeated that kdbus really seems to support a different model, and there isn't likely to be a clean way to support both in one implementation. 

Kay mentioned that while, in his opinion, the interfaces likely can't be shared, he did see some hope for sharing some of the underlying infrastructure. He was particularly keen on the concept of "sealed file descriptors" as being something that Android could make use of. Sealed file descriptors are used when two applications need to share memory by passing a file descriptor; they allow the sender to "seal" the descriptor so that the receiver can be confident that the data won't be later modified by the sender. Kay mentioned the parallels with Android's ashmem, which is the interface Android uses to create shared memory regions that can be shared via file descriptor passing. At this point there was some confusion in the discussion; while the feature does seem useful, it didn't seem to actually mirror how Android currently uses ashmem or shared file descriptors, but it seemed like maybe it would be something that might indeed become more useful once the functionality is upstream. 

I then discussed some of my reasons for being hopeful that Greg would be able to achieve his goal. In particular, there is the issue of binder's complexity and that there really is only one key developer on the Android team who understands the in-kernel binder driver. This makes binder somewhat risky since that developer could be hit by a bus or otherwise stop participating in discussions, making it hard to find someone to continue to maintain binder upstream. Additionally, things like `ioctl()` compatibility support are currently lacking in binder, and, because of its design, it's not easy for 32-bit and 64-bit applications to communicate with each other using binder. Greg noted that everything in kdbus is 64-bit safe, but he also didn't see why binder couldn't be fixed since it's not an official, stable API, which caused some cross talk about how important supporting legacy Android environments is. Colin spoke up to say it really wasn't a big issue, since, when it's needed, the binder library could be made to query the system and do the right thing. 

With that the discussion seemed to be exhausted, with Greg saying we should check back in next year to see if his thinking has changed. 

#### Netfilter extensions

The next topic was netfilter extensions. JP Abgrall from Google's Android team started with a brief background presentation ([slides](<https://docs.google.com/presentation/d/1LpcfW6KgqociD-CYV_gKWRCUI2y4pvyyyeQ8QeBGKu4>)) on the "`xt_qtaguid`" filter he developed, along with some other netfilter modifications Android uses (in particular, `xt_quota2` and `xt_idletimer`) in order to provide per-application network data statistics in its settings interface. 

When these changes were submitted to the community, it was suggested that he look at using the already upstream `xt_owner` and `xt_nfacct` (both of which were merged after `xt_qtaguid` was developed) to provide the same functionality. But JP had a few concerns about the differences between those modules, particularly that using `xt_owner` and `xt_nfacct` would require manipulating the firewall rules during normal operation and would require excessive numbers of rules, both of which could cause performance problems. 

Marcel Holtmann from Intel spoke up to say that he thought the functionality being provided was really nice and that his projects would like to have something similar. But there were some concerns about how it would work with control groups. JP mentioned he had briefly looked at control groups, but they didn't seem useful. This caused some quick debate between Marcel and JP on the particular differences between how Android and desktop environments utilize control groups which I'm not sure clarified much. 

Eric Dumazet, also from Google, but not an Android developer, piped in that he was a networking developer, that the functionality JP wanted with `xt_qtaguid` was already upstream, and that [NFQUEUE](<https://home.regit.org/netfilter-en/using-nfqueue-and-libnetfilter_queue/>) is what they should be using. Since NFQUEUE pushes the packet decisions to user space, this caused a bit of an uproar in the room, as numerous folks were very skeptical that context switching to user space on every packet would be particularly fast or power-efficient. 

Eric reassured everyone that it wasn't a concern, and that the enterprise world uses NFQUEUE for workloads up to some millions of packets per second without trouble. After a bit of contentious back-and-forth with JP, it seemed this issue wouldn't be resolved in the time remaining, and Eric suggested JP come over to his side of the Google campus at a later time to discuss it further. 

Eric also asked about the out-of-tree `xt_quota2` usage and why it was chosen instead of using the already-upstream `xt_quota`. JP mentioned that the in-tree quota code didn't seem useful at all, and the `xt_quota2` code was already implemented. Eric suggested that if the upstream quota didn't work, it should be fixed instead of using `xt_quota2`. It was brought up that Jan Englehardt, the developer of `xt_quota2` had been contacted, and he said the `xt_quota2` code had been rejected by the upstream developers. So being able to better describe the limitations with the upstream quota code will be needed to help persuade upstream maintainers that the functionality in `xt_quota2` is useful. 

#### Android gadget driver and ConfigFS gadget driver

Closing out the microconference session was the Android gadget driver and ConfigFS gadget driver discussion. To try to liven things up a bit, Benoit Goby, of the Google Android team, and Andrzej Pietrasiewicz from Samsung, were seated face-off style in the front of the room for a dramatic showdown. 

Benoit started off with a bit of background on the Android gadget driver. This driver allows Android devices to provide a number of gadget functions, such as support for the picture transfer protocol (PTP), USB mass storage, Ethernet (for tethering), and more over a single USB connection. Additionally, unlike other multi-function gadget drivers already upstream, these multiplexed functions can be dynamically configured at run time. The Android gadget driver patches also provide additional functions that the upstream kernel doesn't yet support, like the media transfer protocol (MTP), the ADB debugger, Android Accessory and Android Audio gadgets. Additionally, the Android gadget driver supports FunctionFS gadgets, which allow gadget functionality to be implemented in user space. In fact, the adbd server has been modified to support the FunctionFS interface, removing the need for an in-kernel ADB function driver. 

Andrzej then similarly described ConfigFS gadget as a dynamically configurable gadget device that allows various gadget functions to be multiplexed. It is different from the Android gadget driver in that it uses [ConfigFS](<https://lwn.net/Articles/148973/>) for setting up and changing the configuration of the various functions. Andrzej talked a bit about the history of the ConfigFS gadget, noting that he originally had taken the Android gadget driver, removed anything that was Android-specific, renamed it Configurable Composite Gadget (CCG) and got it merged into staging. However, when upstream maintainers pushed for use of the ConfigFS interface, the CCG driver was abandoned and Andrzej, along with Sebastian Andrzej Siewior, focused on the ConfigFS gadget implementation. As of 3.10, the ConfigFS gadget is upstream, however it's still missing some desired functionality, like the FunctionFS interface support. 

When asked if he had any issues with the ConfigFS gadget as being a potential upstream replacement for the Android gadget driver, Benoit said he had no objections. Once FunctionFS support and the other Android specific out-of-tree functions, like MTP were merged, it was just a matter of changing Android's user space code to support it. 

Discussion then moved to exactly what the best approach would be for upstreaming the Android-specific gadget functions, like ADB, MTP, Android Accessory and Android audio support. Benoit mentioned again that adbd already supports FunctionFS, so once FunctionFS support is added to the ConfigFS gadget then an ADB function isn't necessary. He also said the MTP implementation could probably be done in user space, but it wasn't clear if that was the best way forward. The Android Accessory function would probably still need to be merged, but the Android Audio interface, he thought, could likely be replaced with a different audio function (though specifically which one and if it was upstream was a little unclear). 

When asked if this all sounded reasonable, Andrzej simply agreed, making this one of the least contentious topics of the day. While it didn't have the lively fireworks hoped for to keep folks awake at the end of a long week, it was a nice and optimistic way to end the planned discussions. 

I'd like to thank everyone for attending and participating in the discussions, as well as Zach Pfeffer and Karim Yaghmour for co-organizing the microconference, and helping with the note taking and reporting.
