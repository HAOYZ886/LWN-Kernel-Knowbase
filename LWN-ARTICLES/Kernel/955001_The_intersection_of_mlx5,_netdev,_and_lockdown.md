---
title: "The intersection of mlx5, netdev, and lockdown"
url: https://lwn.net/Articles/955001/
date: "December 18, 2023"
category: "Development model-Driver merging; Device drivers"
author: "By Jonathan Corbet December 18, 2023"
---

By **Jonathan Corbet**  
December 18, 2023

The NVIDIA Mellanox ConnectX HW family of adapters is a complex beast, supporting networking, InfiniBand, RDMA, and more. As a result, the mlx5 kernel driver that supports this hardware is also complex, as is the interface that it provides to user space. The mlx5 developers have, for a while now, been [proposing](<https://lwn.net/ml/linux-kernel/20231121070619.9836-1-saeed@kernel.org/>) the addition of a new control interface, in the form of a separate virtual device exported by the kernel, that would make vast amounts of debugging information available. This driver has encountered some significant opposition on its way toward the mainline, though, raising a number of questions about appropriate interfaces and when subsystem maintainers have veto power over submissions. 

As described in the cover letter to the patch series, the hardware in question presents a sort of remote-procedure-call interface to the system; that interface is used to control a complex set of operations provided by the device. As developer Saeed Mahameed later [said](<https://lwn.net/ml/linux-kernel/ZXIUysBgNWWZHe0z@x130/>), the device ""is able to expose millions of objects and device states interactively""; that results in ""a complex debugging environment"" where it can be hard to figure out why something is not working correctly. That presents a challenge for both customers and Mellanox support personnel, all of whom need a way to get at information about the state of the device. 

In the past, according to Mahameed, this information was obtained by communicating directly with the device's bus registers using interfaces under `/sys/bus/pci`. Any program that can access the system at that level, of course, is able to compromise the system entirely, so that access would naturally be restricted to privileged users. That, however, is deemed insufficient on systems where [lockdown](<https://lwn.net/Articles/784674/>) policies are in place; PCI access could be used to circumvent lockdown and make persistent changes to the underlying system — exactly the scenario that lockdown is meant to prevent. So, on systems where lockdown is enabled, this approach to obtaining information from mlx5 devices is blocked. 

Lockdown is often seen as a way for consumer-electronics vendors to prevent their customers from controlling the devices that they think they own, and it can certainly be used that way. But it has also found a home in large data centers, where there is a strong motivation to ensure that every machine is running the software that it is supposed to. Those are the same machines that are likely to be running mlx5 hardware, though, leading to a problem for users. How can the necessary information be extracted from a locked-down production system to figure out why something went wrong? 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The proposed answer is the mlx5ctl driver, which provides a restricted interface for the acquisition of debugging information. It also provides the ability to tweak configuration parameters ([about 600 of them](<https://lwn.net/ml/linux-kernel/20231128162413.GP436702@nvidia.com/>), it seems) within the device. With this driver installed, it is possible to talk to the device at a debugging level without direct access to its bus registers, and without running afoul of lockdown protections. 

This driver has run into some concerted opposition, primarily from networking maintainer Jakub Kicinski, who is concerned about the addition of device-specific APIs to the mainline kernel. Modern interfaces have a large number of tunable parameters, and the networking ("netdev") developers have put a lot of effort into creating common interfaces for those parameters whenever possible. That allows the same tools to be used for hardware from multiple vendors, making life easier for both developers and users. Since there is no common API for mlx5-specific parameters, none of those tools will work with mlx5ctl. Kicinski, not liking this state of affairs, has [blocked the driver](<https://lwn.net/ml/linux-kernel/20231127160719.4a8b2ad1@kernel.org/>), saying: ""I don't see how netdev can agree to this driver as long as there is potential for configuring random networking things thru it"". 

Much of the ensuing discussion has circled around the question of what _would_ be an acceptable interface for this data, if mlx5ctl is not it. One possibility that is repeatedly mentioned is the [devlink API](<https://docs.kernel.org/networking/devlink/>), which can be used to access and configure a vast number of parameters on network interfaces. It seems that devlink would work for mlx5, but that there is opposition to using it that way. The networking community does not want to see a proliferation of device-specific devlink parameters, especially if those parameters can be used to configure the operation of an interface. As a result, review of those parameters is required, and getting 600 mlx5-specific parameters past review seems challenging at best. 

If the mlx5 information is not welcome in devlink, then where should it go? Kicinski [mentioned debugfs](<https://lwn.net/ml/linux-kernel/20231128103304.25c2c642@kernel.org/>) at one point in the discussion, and Greg Kroah-Hartman has [suggested](<https://lwn.net/ml/linux-kernel/2023120857-calculus-concerned-cef0@gregkh/>) that approach as well. There are a couple of problems with that idea, though: debugfs is not meant to scale to the amount of data a device can make available, and it is not enabled on locked-down systems. Kicinski has also [suggested](<https://lwn.net/ml/linux-kernel/20231205204855.52fa5cc1@kernel.org/>) just shipping mlx5ctl to customers as an out-of-tree module, which runs counter to the usual advice given by kernel developers. Out-of-tree modules also run afoul of lockdown restrictions, of course; Kicinski [described](<https://lwn.net/ml/linux-kernel/20231207082042.6229868e@kernel.org/>) that restriction as a problem in its own right. 

This discussion appears to be at an impasse; a somewhat frustrated Jason Gunthorpe [described](<https://lwn.net/ml/linux-kernel/20231208133438.GP2692119@nvidia.com/>) the situation this way: 

> Users want an in-tree solution that is compatible with lockdown. A solution that works for all the mlx5 deployment modes (including Infiniband native without netdev) and covers most of the functionality they previously enjoyed with the /sys/../resource based tooling. 
> 
> This series delivers that. 
> 
> Nobody has offered an alternative vision that achieves the same thing. 

The mlx5ctl developers have [raised a related question](<https://lwn.net/ml/linux-kernel/ZXIUysBgNWWZHe0z@x130/>) as well: why do the networking developers have a say over this driver at all? From the mlx5 point of view, it is not a network device. 

> Also I would like to repeat, this is not touching netdev, netdev's policies do not apply to the greater kernel or RDMA, and we have use cases with pure-infiniband/DPU/FPGA cards that have no netdev at all, or other cases with pure virtio instances, and much more. 

This question has not been directly answered other than pointing out that some of these devices do, indeed, have a network interface on them. The fact that mlx5ctl could be used to influence network devices on some hardware is seen as being sufficient to require the approval of the networking developers. 

The question of jurisdiction, for lack of a better word, has come up before in the kernel community. As a recent example, consider [AI accelerators](<https://lwn.net/Articles/870418/>), which look a lot like graphics coprocessors without a display controller. The graphics community has spent years developing and enforcing requirements, including the availability of a free user-space implementation of each device's functionality, on GPU drivers. AI accelerators were being upstreamed via a separate path where such requirements were not enforced, leading to protests from the graphics community. In the end, after extensive discussions, an accommodation was reached that brought AI accelerators under a similar set of rules. 

This case looks similar; networking developers see an interface to network interfaces that does not follow that subsystem's rules, and they worry that their hard work to prevent a proliferation of device-specific configuration APIs will be undermined. The mlx5ctl developers, instead, feel that they are being prevented from merging a proper upstream implementation of needed functionality by developers from an only marginally related subsystem. The end result is a fair amount of frustration and little apparent progress toward a solution. 

Eventually, it would seem, some sort of understanding will need to be reached here. What that will look like is not clear at this point; getting there will require the people who are closest to the problem to find a way to work together toward a solution that addresses the concerns of both sides. The kernel community tends to find such a solution eventually, but the road to that destination can be bumpy at times.
