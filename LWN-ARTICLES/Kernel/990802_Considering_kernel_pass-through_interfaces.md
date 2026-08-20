---
title: "Considering kernel pass-through interfaces"
url: https://lwn.net/Articles/990802/
date: "September 20, 2024"
category: "Development model-Driver merging; Device drivers"
author: "By Jonathan Corbet September 20, 2024 Maintainers Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 20, 2024

* * *

[Maintainers Summit](<https://lwn.net/Articles/990740/>)

The kernel normally sits firmly between user space and the system's peripheral devices, and provides a standard interface to those devices. At times, though, a more direct interface to a device is desired — but such interfaces can be controversial. At the 2024 Maintainers Summit, the assembled developers considered a specific case — the proposed [fwctl subsystem](<https://lwn.net/Articles/969383/>) — as well as the role of such drivers in general. 

[![\[Dan Williams\]](https://static.lwn.net/images/conf/2024/ms/DanWilliams-sm.png)](<https://lwn.net/Articles/990847/>) Fwctl comes out of [a longstanding disagreement](<https://lwn.net/Articles/955001/>) over a driver initially called mlx5ctl; its purpose is to allow a user-space utility to adjust any of hundreds of tunable parameters within mlx5 devices (which implement InfiniBand, RDMA, and other protocols). Proponents say that this interface is necessary to configure the device properly; opponents say that it is a way to bypass the kernel and that device-independent interfaces should be designed for this task instead. The discussion went on over the course of a year or so, with no resolution in sight. 

The session was led by Dan Williams and fwctl developer Jason Gunthorpe. Williams started by saying that he had two objectives in mind. One was to try to heal the community; he has been watching two developers he respects in conflict and the conversation deteriorating over time. Additionally, there were junior developers finding themselves caught in the middle who might just end up leaving the community. But Williams also had a more personal goal: he, too, is working on an API for device provisioning and feels the need to allow direct access to vendor commands to get the job done. 

The argument against fwctl, he said, is that this kind of configuration should be done with the network subsystem's [devlink interface](<https://docs.kernel.org/networking/devlink/>) instead; fwctl is seen as a shortcut to avoid creating a proper interface. Allowing fwctl, it is feared, would reduce the motivation to improve devlink. Proponents of fwctl, instead, say that devlink does not provide the needed functionality, and that insisting on it is forcing these interfaces to be maintained outside of the mainline instead. 

#### Inspired by lockdown

Gunthorpe explained that the kernel [lockdown](<https://man7.org/linux/man-pages/man7/kernel_lockdown.7.html>) feature disables access to `/dev/mem`, which is the interface by which these devices had traditionally been controlled. This interface provides direct access to the system's I/O memory, which presents no end of potential security problems. Fwctl is an attempt to keep the configuration software working in a locked-down world. He is well familiar with the devlink interface, but there is no standard behind the configuration of these complex devices; every manufacturer creates its own set of knobs that is hard to bring into a common interface. 

[![\[Jason Gunthorpe\]](https://static.lwn.net/images/conf/2024/ms/JasonGunthorpe-sm.png)](<https://lwn.net/Articles/990847/>) Even so, the original plan for mlx5ctl had been to use devlink, but that quickly led to having to argue about 300 different parameters, each of which had to be standardized and approved independently. Additionally, devlink does not provide access to debugging information, while mlx5ctl and fwctl provide access to that sort of data. There is no alternative proposal out there, he said, that provides the needed access. 

Devlink, he said, has its origins in the "WiFi debacle", where every driver provided a different configuration interface. It cleaned up that situation and led to the conclusion that providing direct access to device-configuration interfaces was a mistake. That conclusion made sense in the networking context, but there are no standards for these complex, server-oriented devices, so devlink is not a good fit. 

Williams said that he would like the group to agree that non-generic device commands exist, and that users need access to those commands. For the [CXL](<https://en.wikipedia.org/wiki/Compute_Express_Link>) devices he works with, the policy has been that no such vendor commands would be enabled, that any needed functionality must be incorporated into the standards instead. "That has happened zero times", he said; making it harder to solve problems does not force vendors to come and talk to us. A different policy is called for here, he said. Gunthorpe added that generic interfaces are the right solution for WiFi configuration, but they are a poor match to key-value stores buried in device firmware. 

There are, Williams continued, classes of device commands that the kernel just does not care about. These include device-specific configuration and access to debugging information. He asked the group to agree that, in such cases, the kernel should not stand in the way; the alternative is that vendors just push distributors to ship out-of-tree code instead. 

#### Security boundaries

Dave Airlie worried that such interfaces could facilitate the compromise of the whole system; almost all firmware has that ability (by writing arbitrary system memory, for example) somewhere. Since locked-down systems are involved, security is clearly a concern; given the low quality of most firmware implementations, he is worried about providing this kind of access. Will vendors provide assurances that their systems cannot be used to compromise the kernel? 

Gunthorpe replied that devlink can be used now to flash new firmware, so all bets are off when using that interface too. Linus Torvalds, more bluntly, said that "lockdown is a joke", a public-relations feature that lets kernel developers make a show of being careful. With regard to fwctl, he said that he does not see what the argument is about; forcing the use of devlink has no benefits for either the kernel or the hardware vendors. He added that, if somebody is running as root, they can do what they want with the hardware; "we are not doing DRM". 

Ted Ts'o said "if you don't trust the hardware, you might as well just go home". Arnd Bergmann asked why users wanting to run these configuration utilities don't just turn off lockdown. Gunthorpe answered that customers (especially governments) often require it to be enabled. Fwctl was created partially to support this kind of deployment; it includes a long list of rules on what commands are allowed to do, so that they do not compromise system security. 

Damien Le Moal agreed that existing commands provide plenty of ways for somebody to damage their system; if they do so, it is their fault. But, he said, a device driver's job is to configure hardware properly; he wondered why this additional interface was needed. Gunthorpe answered that modern devices are hugely complex and must be configured to work within the environment in which they are used. Vendors can ship special configurations suited to the needs of the largest customers — the Googles and Metas of the world. Smaller customers, though, must customize their devices in the field; that is where this kind of interface is needed. 

Kees Cook agreed that, in the end, there is no alternative to trusting the hardware. Existing ways of configuring hardware using `/dev/mem` are opaque; the fwctl interface is better, he said. Gunthorpe agreed that fwctl is far better for the reverse-engineering of hardware; it also makes it easier for the kernel to block or modify specific commands if they turn out to be problematic. Williams added that vendors can normally be trusted to abide by the system's security boundaries. 

Ts'o pointed out that the `SG_IO` operation (which allows arbitrary commands to be sent to SCSI devices) has been supported by the kernel since the early days; it provides the same capabilities as fwctl. Perhaps, he said, `SG_IO` is only "grandfathered because CD burners", but it would be good to know what the policy is. Having distributors applying out-of-tree patches seems like a worse outcome than just including fwctl, he said. 

Gunthorpe said that he is trying to create a general policy for this kind of interface. Torvalds said that users have to be able to run device-specific commands; `SG_IO` is a good example of the sort of capability that the kernel has always supported. It is fine for the kernel to apply a root-only policy to commands it does not recognize, but he has no interest in saying that the owner of a machine cannot manage their hardware. 

The rule, he said, is that developers should try to prevent each device from implementing its own pass-through command; instead, there should be some sort of baseline (such as fwctl) for this access. Permissible commands should only change the device in question, but touch no other part of the system. There should be "no random DMA" operations. In the end, though, the kernel has to trust the hardware. 

Will Deacon asked how fwctl would interact with common interfaces; will it be possible to spot commands that are common between devices and standardize them? Gunthorpe answered that common interfaces provide a better interface for users. Some of that needs to be provided by user-space utilities, though; it is easier to create shared interfaces there. 

Cook said that having arbitrary applications accessing device memory via `/dev/mem` is bad; there is no way for the kernel to impose a policy in that setting. Torvalds answered: "we call that X11". Cook said that he wants to see documentation of the commands supported by a device and what they do. Airlie answered that there will only be a single command, analogous to `SG_IO`, that passes operations through to the device. Torvalds said that this interface is still better than using `/dev/mem`. 

#### Cross-subsystem disagreement

Williams raised another aspect of this debate — that of cross-subsystem nacks. The fwctl patches have been blocked by the networking subsystem maintainer, even though that work is not a part of his domain. How far, Williams asked, can a maintainer's veto extend beyond their own subsystem? Airlie answered that this was Torvalds's problem. Gunthorpe said that there is a precedent for how to respond to this sort of block: the RDMA subsystem was started as a response to the blocking of support for TCP offload engines in the networking subsystem. That block was the right decision for networking, but there is still a place for TCP offloading in the kernel. RDMA is widely used and supported by open-source software now; it is, he said, a great success story. Fwctl could be a similar story. 

There seemed to be a clear consensus in the room that work on fwctl should proceed and find its way into the mainline; Gunthorpe was asked about how he planned to do that. He answered that he likes to see three independent users before merging a new subsystem; that helps to show that the interfaces are correct. A proposed third fwctl user had showed up in his inbox that morning, adding to the existing RDMA and CXL users. For now, he plans to focus on getting the drivers into good shape, and expects to send out a pull request in roughly six months. An implementation will ship to Mellanox users sooner, though. 

Six months may seem like a long time; Gunthorpe said that he has been taking it slowly and carefully because of the pushback he has been receiving. He respects the people who have opposed this work and wants to show that respect by doing the job properly. Even so, he expects the nack from the networking subsystem to persist; it is "the right position" for that subsystem to have, he said, but would like to have some peace and get this work done. The session closed with him saying that he would have further discussions with the developers involved.
