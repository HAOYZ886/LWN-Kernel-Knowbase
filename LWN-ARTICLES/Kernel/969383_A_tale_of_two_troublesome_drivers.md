---
title: A tale of two troublesome drivers
url: https://lwn.net/Articles/969383/
date: "April 12, 2024"
category: "Development model-Driver merging; Device drivers"
author: "By Jonathan Corbet April 12, 2024"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 12, 2024

The kernel project merges dozens of drivers with every development cycle, and almost every one of those drivers is entirely uncontroversial. Occasionally, though, a driver submission raises wider questions, leading to lengthy discussion and, perhaps, opposition. That is currently the case with two separate drivers, both with ties to the networking subsystem. One of them is hung up on questions of whether (and how) all device functionality should be made available to user space, while the other has run into turbulence because it drives a device that is unobtainable outside of a single company. 

#### mlx5ctl and fwctl

The mlx5ctl driver is not a new problem; it was [covered here](<https://lwn.net/Articles/955001/>) in December 2023. In short: this driver implements a transport channel allowing user space to query and manipulate parameters on an mlx5 device (which provides a range of networking, RDMA, and InfiniBand functionality), crucially without any understanding of those parameters on the kernel's part. Proponents say that this driver is needed to provide users with the access needed to configure and debug their hardware, especially on locked-down systems where other methods of talking directly to the hardware are unavailable. Opponents see it as a way of circumventing the normal development process that governs how device parameters are exported to user space. 

Saeed Mahameed posted [a new version of the mlx5ctl patch series](<https://lwn.net/ml/linux-kernel/20240207072435.14182-1-saeed@kernel.org/>) at the beginning of February, saying: ""We continue to think that mlx5ctl is reasonable and aligned with the greater kernel community values"". Christoph Hellwig [responded](<https://lwn.net/ml/linux-kernel/Zcx53N8lQjkpEu94@infradead.org/>) with an ack and a complaint about the ""subsystem maintainer overreach"" that has blocked the merging of this driver. Networking maintainer Jakub Kicinski [agreed](<https://lwn.net/ml/linux-kernel/20240214074832.713ca16a@kernel.org/>) that ""overreach is unfortunate"", but also maintained the position that this driver should not be merged: ""We have a clear rule against opaque user space to FW [firmware] interfaces"". Beyond that, there was not a lot of other discussion on the submission at that time. 

At the beginning of March, though, Jason Gunthorpe posted [a proposal](<https://lwn.net/ml/linux-kernel/20240304160237.GA2909161@nvidia.com/>) for a new subsystem called "fwctl" that would be the home for drivers like mlx5ctl. Modern devices, he wrote, tend to come with a large set of tunable parameters controlling many aspects of their functionality; these parameters need to be made accessible by user space if users are to be able to use their hardware. 

> fwctl's purpose is to define a common set of limited rules, described below, that allow user space to securely construct and execute RPCs inside device FW. The rules serve as an agreement between the operating system and FW on how to correctly design the RPC interface. As a uAPI the subsystem provides a thin layer of discovery and a generic uAPI to deliver the RPCs and collect the response. It supports a system of user space libraries and tools which will use this interface to control the device. 

The proposal goes into some detail on the types of functionality that will be made available via fwctl interfaces. It also covers the functionality that _cannot_ be provided, including the ability to DMA to arbitrary memory, manipulate kernel memory or subsystems outside of the driver itself, or provide functionality, such as sending a network packet, that should be handled by another subsystem. 

As before, the primary opposition (to both mlx5ctl and fwctl) came from Kicinski. He [described](<https://lwn.net/ml/linux-kernel/20240322135826.1c4655e2@kernel.org/>) the justification for this work as ""smoke and mirrors"", saying it was a way for manufacturers to ""hide as much as possible of what you consider your proprietary advantage in the 'AI gold rush'"". Complex hardware, he [said](<https://lwn.net/ml/linux-kernel/20240322152924.64be7ec4@kernel.org/>), does not need a backdoor to talk to the firmware without the kernel's mediation; he cited the network interfaces used at Meta (his employer) as an example. He [questioned](<https://lwn.net/ml/linux-kernel/20240322154027.5555780a@kernel.org/>) whether the restrictions on fwctl drivers would be enforced, and said that the conversation did not appear to be going anywhere useful: 

> Or should we go for another loop of me talking about openness and building common abstractions, and vendors saying how their way of doing basic configuration is so very special, and this is just for debug and security and because others. 
> 
> There's absolutely no willingness to try and build a common interface here. 

Kicinski has repeatedly said that this functionality should be provided via an API like [devlink](<https://docs.kernel.org/networking/devlink/>), where parameters are exposed after a community review that is, among other things, intended to force consistency between hardware from different manufacturers. He [complained](<https://lwn.net/ml/linux-kernel/20240403170149.7d2b8f2b@kernel.org/>) that his offer to quickly review proposed devlink knobs had been ignored by the vendors looking for interfaces like fwctl. 

On the other side, David Ahern [asserted](<https://lwn.net/ml/linux-kernel/1cd2a70c-17b8-4421-b70b-3c0199a84a6a@kernel.org/>) that fwctl _is_ the common interface that Kicinski is looking for. Gunthorpe [said](<https://lwn.net/ml/linux-kernel/20240402184055.GP946323@nvidia.com/>) that all complex devices require hardware-specific tooling to configure them to the customer's needs. The only reason Meta does not need such tools is that, as a large customer, it is able to receive its hardware preconfigured from the vendor; smaller customers do not receive that level of service. Vendors have been providing these tools for years, he said; fwctl is just a way to provide a common interface for them. 

The problem with the devlink approach, Gunthorpe [added](<https://lwn.net/ml/linux-kernel/20240404122338.GI1723999@nvidia.com/>), is that, beyond the slow and painful nature of the process, it is guaranteed to fail. To be useful, an interface must be able to work with all of the parameters provided by the device: 

> As far as configuration/provisioning goes, it is really all or nothing. 
> 
> If a specific site can configure only 90% of the stuff required because you will NAK the missing 10% it then it is still not usable and is a wasted effort for everyone. 
> 
> You have never shown that there is a path to 100% with your approach to devlink. In fact I believe you've said flat out that 100% is not achievable. 

Kicinski [was not receptive](<https://lwn.net/ml/linux-kernel/20240404074850.19ecd52e@kernel.org/>) to this argument, though, calling many of the knobs ""hacks and lazy workarounds"". 

As of this writing, this discussion does not appear to be any closer to a resolution than it was in December. The positions taken have only hardened over time. In the end, the fate of this driver (or for a future fwctl subsystem) may well depend on whether Linus Torvalds is willing to allow a networking maintainer to block the merging of a driver that is, by most accounts, independent of the networking subsystem. 

#### A network interface for one

At the beginning of April, Alexander Duyck posted [a driver called "fbnic"](<https://lwn.net/ml/netdev/171217454226.1598374.8971335637623132496.stgit@ahduyck-xeon-server.home.arpa/>) for a custom network interface card that is used only within Meta. That prompted [an immediate question](<https://lwn.net/ml/netdev/Zg6Q8Re0TlkDkrkr@nanopsycho/>) from Jiri Pirko, who wondered why the community needs a driver for a device that nobody is able to acquire. Duyck [responded](<https://lwn.net/ml/netdev/CAKgT0Uf8sJK-x2nZqVBqMkDLvgM2P=UHZRfXBtfy=hv7T_B=TA@mail.gmail.com/>) that upstreaming the driver would make maintenance easier, that it would make it easier to introduce new networking features implemented in the driver, and that the company might someday open some of the hardware information as well. Pirko [was unimpressed](<https://lwn.net/ml/netdev/Zg7JDL2WOaIf3dxI@nanopsycho/>) and said that the driver should not be merged. 

Duyck [called](<https://lwn.net/ml/netdev/CAKgT0Ufgm9-znbnxg3M3wQ-A13W5JDaJJL0yXy3_QaEacw9ykQ@mail.gmail.com/>) this reasoning ""arbitrary and capricious"". The driver will have a lot of users at Meta, he said. There have been other proprietary devices added to the kernel in the past; the [Intel IDPF driver](<https://docs.kernel.org/networking/device_drivers/ethernet/intel/idpf.html>) was mentioned as an example elsewhere in the conversation. Drivers also often show up for devices that are not yet for sale, and may never make it to the market. To reject the driver, he said, is an accusation of ""some lack of good faith"" from Meta. 

Kicinski [tried to redirect the discussion](<https://lwn.net/ml/netdev/20240404132548.3229f6c8@kernel.org/>) somewhat, saying that he did not want to be in the position of judging the "good faith" of companies. The community, he said, had to make its decision based on the interest of the project and the broader user base. He did not say, then, whether he thought the driver should be merged or not. Others, though, such as [John Fastabend](<https://lwn.net/ml/netdev/660f22c56a0a2_442282088b@john.notmuch/>) and [Paolo Abeni](<https://lwn.net/ml/netdev/678f49b06a06d4f6b5d8ee37ad1f4de804c7751d.camel@redhat.com/>), argued that fbnic appeared to be good code, and that in any case it is only a network-interface driver with no potential to harm the rest of the kernel, so there is no reason to keep it out. 

Gunthorpe, while not arguing against the merging of fbnic, [raised some concerns](<https://lwn.net/ml/netdev/20240405122646.GA166551@nvidia.com/>). There is a strong feeling that code should not be merged solely for the purpose of supporting proprietary user-space code, he said, and ""this submission is clearly blurring that line"". That could, he [said](<https://lwn.net/ml/netdev/20240405190209.GJ5383@nvidia.com/>), lead to problems in the future as more features are added to the driver. 

There was a brief turn in the conversation when Andrew Lunn [referred](<https://lwn.net/ml/netdev/d0bd66c7-13d2-40a3-8d7d-055ac0234271@lunn.ch/>) to the mlx5ctl discussion and asked Duyck to show that a separate firmware-tuning driver would not be required for this device. Kicinski [said](<https://lwn.net/ml/netdev/20240404083752.5c81d369@kernel.org/>) that showing that would not change anybody's mind. Ahern [suggested](<https://lwn.net/ml/netdev/a1e5ccd3-1a9b-4d78-8216-2274e1222bb8@kernel.org/>) that, in the future, when ""the inevitable production problems"" show up, a separate, mlx5ctl-like driver may well become necessary. 

Perhaps the biggest concern, though, was [expressed](<https://lwn.net/ml/netdev/20240404165000.47ce17e6@kernel.org/>) by Kicinski: what happens if changes elsewhere in the kernel break the driver, creating a regression for its users? Since the community as a whole cannot test the driver, such breaks could be hard to avoid and even harder to fix; that could lead to kernel changes being reverted. In such a situation, a private driver like fbnic could impede kernel development in general. 

For that reason, though Kicinski eventually [concluded](<https://lwn.net/ml/netdev/20240409135142.692ed5d9@kernel.org/>) that ""there's broad support for merging the driver"", he also said that there needs to be a slightly different set of rules governing drivers for private devices. These would include ""weaker 'no regression' guarantees"" and an expectation that the driver maintainers will participate actively in efforts to refactor subsystem interfaces. In the absence of such participation, a driver for a private device could be removed from the kernel. Pirko eventually [agreed](<https://lwn.net/ml/netdev/ZhasUvIMdewdM3KI@nanopsycho/>) that, if the driver were to be marked as belonging to this new regime (which would have to be documented), it ""would be ok to let this in"". 

So the fbnic driver seems likely to be merged in the end. The same may eventually be true of mlx5ctl in some form as well. The Linux kernel did not get to the position it is in by refusing to let users access the full capabilities of their hardware, and it seems unlikely to adopt such a policy now. A more difficult prospect, though, is to guess how many more lengthy discussions will be required to reach that decision.
