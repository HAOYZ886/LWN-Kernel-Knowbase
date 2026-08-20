---
title: A filesystem for namespaces
url: https://lwn.net/Articles/877308/
date: "December 3, 2021"
category: Containers; Namespaces
author: "By Jonathan Corbet December 3, 2021"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 3, 2021

It is natural, when looking at the kernel development process, to focus on patches that find their way to acceptance and become a part of future kernels. But there can be value in looking at work that doesn't clear the bar; in failing, these patches often reveal things about the kernel and the community that creates it. Such is the case with the [proof-of-concept namespacefs patch series](<https://lwn.net/ml/linux-kernel/20211118181210.281359-1-y.karadz@gmail.com/>) recently posted by Yordan Karadzhov. One should not expect to see namespacefs in a future kernel but, in failing, this work showed a real use case and why it is hard to satisfy that use case in the kernel. 

Namespacefs is, as one might expect, a virtual filesystem implemented by the kernel. Its job is to display the hierarchy of namespaces running on the system; this information reflects the hierarchy of containers that are running. By using namespacefs, administrators can more readily see what is happening on their systems; it is also meant to facilitate complicated use cases like tracing multiple containers and watching how they interact. 

The initial implementation was limited to the [PID](<https://lwn.net/Articles/531419/>) and [time](<https://lwn.net/Articles/766089/>) namespaces. One can use it to traverse the hierarchy of PID namespaces (time namespaces are not hierarchical) and obtain the list of processes running in each. Other types of namespaces are not supported in this posting, but the intent was seemingly to add that support in a future version if namespacefs looked like the right solution to the problem. 

As Karadzhov wrote: 

> Being able to see the structure of the namespaces can be very useful in the context of the containerized workloads. This will provide universal methods for detecting, examining and monitoring all sorts of containers running on the system, without relaying on any specific user-space software. 

Much of this information is available in user space now in the form of directories under `/proc`, but there are some missing pieces and that information is not organized in a way that shows the actual namespace hierarchy. Container-orchestration systems can also provide a view of the containers they manage, of course, but they don't provide a solution for the general case. Namespacefs was meant to make this information readily available regardless of which orchestration systems are in use. 

There were a few objections raised to this work, starting with the fact that a namespace's entry in namespacefs needs to have a name. There are currently no names associated with namespaces, so namespacefs uses the number of the inode that is attached to each namespace inside the kernel. Eric Biederman was quick to [criticize](<https://lwn.net/ml/linux-kernel/87a6i1xpis.fsf@email.froward.int.ebiederm.org/>) that approach, saying: ""It is not correct to use inode numbers as the actual names for namespaces"". He went on to say that there was nothing else that could be used as names for namespaces either, and that the entire idea was unworkable. 

There are, it seems, a couple of problems with using inode numbers as names for namespaces. One of those, which Biederman [spelled out](<https://lwn.net/ml/linux-kernel/87pmqxuv4n.fsf@email.froward.int.ebiederm.org/>) later, is that there is no way to recreate the namespace hierarchy at a later time with the same names. That, he said, would break any system that uses, for example, [CRIU](<https://criu.org/Main_Page>) to checkpoint and restart containers, perhaps as part of a live-migration scheme. The only way to handle this properly, he said, is to create a namespace for namespace names, and that has proved to be a hard problem in the past. 

The CRIU issue is only relevant if containers that may be checkpointed will use namespacefs. As both [Karadzhov](<https://lwn.net/ml/linux-kernel/1613eb28-f5d2-6ede-b0a1-f48c5ce240fc@gmail.com/>) and [Steve Rostedt](<https://lwn.net/ml/linux-kernel/20211118143634.3f7d43e9@gandalf.local.home/>) pointed out, that is unlikely; the whole point of namespacefs is to show the situation on a specific machine. There is no reason for anybody to want to move namespacefs — or any container making use of namespacefs — across machines or even to checkpoint it. It is, of course, hazardous to assume that nobody will want to use a feature in a certain way in the future but, in the absence of a surprising use case, the naming problem may not be an issue in actual use. 

An arguably deeper problem, though, is that namespacefs can be seen as an attempt to recognize containers in the kernel, but the kernel has (by design) no concept of a "container". The kernel, instead, provides a set of pieces that user-space systems can assemble into varying types of containers. The namespacefs patches use PID namespaces as the objects around which the hierarchy is built and ignore [user namespaces](<https://lwn.net/Articles/532593/>) entirely. Biederman, in his initial response, criticized that decision, saying that ""there is definitely no meaningful hierarchy without the user namespace"". Not all containers use user namespaces, though, and those namespaces lack the process-ID information that Karadzhov's patch was meant to expose in the first place. 

But, as James Bottomley [pointed out](<https://lwn.net/ml/linux-kernel/f141c401560d90a546968514c6cfc63d7fdb8e00.camel@HansenPartnership.com/>), not all containers use PID namespaces either. Trying to identify containers without PID namespaces in namespacefs is not going to lead to much joy. 

The end result is that it appears difficult to implement something like namespacefs in the kernel without introducing some sort of concept of what a container is. There is no more appetite for doing that now than [there has been in past years](<https://lwn.net/Articles/723561/>); the lack of a container abstraction in the kernel is seen as having enabled a great deal of innovation on the user-space side. For this reason alone, namespacefs would be a hard sell in the kernel community. 

It also appears, though, that it should be possible to get the required information entirely in user space by digging through a lot of `/proc` files. If there is information that is missing, it can be added to `/proc` rather than introducing an entirely new filesystem. So that is [the approach that Karadzhov will take](<https://lwn.net/ml/linux-kernel/e94c2ba9-226b-8275-bef7-28e854be3ffa@gmail.com/>) to solve this problem. Another proof of concept will be put together to show how it would work. 

If that implementation turns out to be difficult or impossible to do efficiently, then there might be an argument for reconsidering namespacefs. Otherwise, though, a mechanism like namespacefs seems unlikely to make it into the kernel. That particular effort may not have led directly to the desired result, but it did create a discussion that coalesced on a seemingly better solution and, in the process, highlighted some of the constraints brought by the kernel's lack of a container concept. A reluctance to implement policy is generally a good thing, but it can end up making certain kinds of problems harder for users to solve.
