---
title: Memory overcommit in containerized environments
url: https://lwn.net/Articles/931658/
date: "May 15, 2023"
category: "Memory management-Virtualization"
author: "By Jonathan Corbet May 15, 2023 LSFMM+BPF"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 15, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Overcommitting memory is a longstanding tradition in the Linux world (and beyond); it is rare that an application uses all of the memory allocated to it, so overcommitting can help to improve overall memory utilization. In situations where memory has been overcommitted, though, it may be necessary to respond quickly to ensure that applications have the memory they actually need, even when those needs change. At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), T.J. Alumbaugh (in the room) and Yuanchu Xie (remotely) presented a new mechanism intended to help hosts provide containerized guests with the memory resources they need. 

Xie started by pointing out that, while containers are most often seen as a server-side technology, both server and client applications are often run in containerized environments. Those two types of workloads can vary in their execution, though; server applications tend to run constantly and predictably, while clients can be more bursty as they respond to user interactions. For server applications, the focus tends to be on reliability, while clients aim for responsiveness. Proper management of overcommitted memory is important in both cases. 

Providing the memory resources that a containerized application needs — and no more — requires understanding what that application's working set is at any given time. The working set can be seen as a sort of histogram, where pages of memory are placed in bins according to some metric, usually the time of last access or some estimate of coldness. These bins can take the form of generations in the [multi-generational LRU](<https://lwn.net/Articles/856931/>) or the traditional active/inactive-list mechanism used by the kernel for years. Indeed, sometimes the classification of pages can even be done in user space. 

One way of controlling the memory available to a container is a balloon device, which can allocate memory within the container (thus "inflating") and return that memory to the host if a container's memory needs to shrink. The balloon can be deflated to give a container more memory. The work under discussion is aimed at collecting working-set data and providing it to the host by way of the balloon device. The host can then use this information to respond to changes in memory use. 

[![\[T.J. Alumbaugh\]](https://static.lwn.net/images/conf/2023/lsfmm/TJAlumbaugh-sm.png)](<https://lwn.net/Articles/931744/>) Alumbaugh took over to talk about the notification mechanism. In short, the balloon driver within the container will be informed (by whatever mechanism is employed to monitor memory use) when a new working-set report is available, via a shrinker-like callback interface. That information can then be passed up to the host, which will use it to implement its resource-management policies. Actions taken in response to working-set reports can include setting control-group limits, or changing the balloon size in specific virtual machines. 

[Patches implementing this mechanism](<https://lwn.net/ml/linux-mm/20230509185419.1088297-1-yuanchu@google.com/>) have been posed to the mailing lists, he said, and an associated QEMU patch set will be posted soon. Google's [crosvm](<https://github.com/google/crosvm>) virtual-machine monitor has already gained support for working-set reports transmitted in this way, and there have been discussions on adding it to the Virtio specification as well. 

The only response to the presentation was a comment from David Hildenbrand, who expressed his dislike for balloon drivers as a way of controlling memory resources. Without care, balloon inflation can create out-of-memory situations, which is rarely the desired result. It is better, he said, to use free-page reporting to the host, which can respond by telling guests to adjust their working-set sizes. That provides guests with the flexibility they need to avoid out-of-memory problems. The core idea is the same as what had been presented, he said, but the mechanism is different.
