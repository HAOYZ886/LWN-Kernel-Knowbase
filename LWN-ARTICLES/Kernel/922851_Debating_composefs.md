---
title: Debating composefs
url: https://lwn.net/Articles/922851/
date: "February 16, 2023"
category: "Filesystems-composefs"
author: "By Jonathan Corbet February 16, 2023"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 16, 2023

When LWN [looked at the composefs filesystem](<https://lwn.net/Articles/917097/>) in December, we reported that there had been "little response" to the patches. That is no longer the case. Whether composefs (or something like it) should be merged has become the subject of an extended debate; at its core, the discussion is over just how Linux should support certain types of container workloads. 

Composefs is an interesting sort of filesystem, in that a mounted instance is an assembly of two independent parts. One of those, an "object store", is a directory tree filled with files of interest, perhaps with names that reflect the hash of their contents; the object store normally lives on a read-only filesystem of its own. The other is a "manifest" that maps human-readable names to the names of files in the object store. Composefs uses the manifest to create the filesystem that is visible to users while keeping the object store hidden from view. The resulting filesystem is read-only. 

This mechanism is intended to create the system image for containers. When designing a container, one might start with a base operating-system image, then add a number of layers containing the packages needed for that specific container's job. With composefs, the object store contains all of the files that might be of interest to any container the system might run, and the composition of the image used for any specific container is done in the manifest file. The result is a flexible mechanism that can mount a system image more quickly than the alternatives while allowing the object store to be verified with [fs-verity](<https://www.kernel.org/doc/html/latest/filesystems/fsverity.html>) and shared across all containers in the system. 

#### The v3 discussion

[Version 3 of the composefs patches](<https://lwn.net/ml/linux-kernel/cover.1674227308.git.alexl@redhat.com/>) was posted in late January; it included a number of improvements requested by reviewers of the previous versions. Amir Goldstein [was not entirely happy](<https://lwn.net/ml/linux-kernel/CAOQ4uxgGc33_QVBXMbQTnmbpHio4amv=W7ax2vQ1UMet0k_KoA@mail.gmail.com/>) with the work that had been done, though; he suggested that, rather than proposing composefs, the developers (Alexander Larsson and Giuseppe Scrivano) should put their efforts into improving the existing kernel subsystems (specifically [overlayfs](<https://docs.kernel.org/filesystems/overlayfs.html>) and the [EROFS filesystem](<https://docs.kernel.org/filesystems/erofs.html>)) instead. 

A long discussion of the value (or lack thereof) of composefs followed. There are currently two alternatives to composefs that are used for container use cases: 

  * At container boot time, the system image is assembled by creating a large set of symbolic links to the places where the target files actually live. This approach suffers from a lengthy startup time; at least one system call is required to create each of thousands of symbolic links, and that takes time. 
  * The container is given a base image on a read-only filesystem; overlayfs is then used to overlay one or more layers on top to create the container-specific image. 

The consensus seems to be that the symbolic-link approach, due to its startup cost, is not a viable alternative to composefs. Goldstein and others do think that the overlayfs approach could be viable, though, perhaps with a few changes to that filesystem. The composefs developers are not so sure. 

#### Considering overlayfs

One current shortcoming with overlayfs, Scrivano [said](<https://lwn.net/ml/linux-kernel/87ilh0g88n.fsf@redhat.com/>), is that, unlike composefs, it is unable to provide fs-verity protection for the filesystem as a whole. Any changes that affect the overlay layer would bypass that protection. Larsson [described](<https://lwn.net/ml/linux-kernel/1ea88c8d1e666b85342374ed7c0ddf7d661e0ee1.camel@redhat.com/>) an overlayfs configuration that could work (with some improvements to overlayfs), but was unenthusiastic about that option: 

> However, the main issue I have with the overlayfs approach is that it is sort of clumsy and over-complex. Basically, the composefs approach is laser focused on read-only images, whereas the overlayfs approach just chains together technologies that happen to work, but also do a lot of other stuff. 

Among other things, he said, the extra complexity in the overlayfs solution leads to worse performance. The benchmark he used to show this was to create a filesystem using both approaches, then measure the time required to execute an "`ls -lR`" of the whole thing. In the cold-cache case, the overlayfs solution took about four times as long to run; the performance in the warm-cache case was more comparable, but composefs was still faster. 

Goldstein [strongly contested](<https://lwn.net/ml/linux-kernel/CAOQ4uxinsBB-LpGh4h44m6Afv0VT5yWRveDG7sNvE2uJyEGOkg@mail.gmail.com/>) the characterization of the overlayfs solution; he also started an extended sub-thread on whether the "`ls -lR`" benchmark made sense. He [added](<https://lwn.net/ml/linux-kernel/CAOQ4uxhGX9NVxwsiBMP0q21ZRot6-UA0nGPp1wGNjgmKBjjBBA@mail.gmail.com/>): ""I see. composefs is really very optimized for ls -lR. Now only need to figure out if real users start a container and do ls -lR without reading many files is a real life use case."" Dave Chinner [jumped in](<https://lwn.net/ml/linux-kernel/20230125041835.GD937597@dread.disaster.area/>) to defend this test: 

> Cold cache performance dominates the runtime of short lived containers as well as high density container hosts being run to their container level memory limits. `ls -lR` is just a microbenchmark that demonstrates how much better composefs cold cache behaviour is than the alternatives being proposed. 

He added that he has used similar tests to benchmark filesystems for many years and has never had to justify it to anybody. Larsson, meanwhile, [explained](<https://lwn.net/ml/linux-kernel/b8601c976d6e5d3eccf6ef489da9768ad72f9571.camel@redhat.com/>) the emphasis that is being placed on this performance metric (or "key performance indicator" — KPI) this way: 

> My current work is in automotive, which wants to move to a containerized workload in the car. The primary KPI is cold boot performance, because there are legal requirements for the entire system to boot in 2 seconds. It is also quite typical to have shortlived containers in cloud workloads, and startup time there is very important. In fact, the last few months I've been primarily spending on optimizing container startup performance (as can be seen in the massive improvements to this in the upcoming podman 4.4). 

Goldstein finally [accepted](<https://lwn.net/ml/linux-kernel/CAOQ4uxiPLHHnr2=XH4gN4bAjizH-=4mbZMe_sx99FKuPo-fDMQ@mail.gmail.com/>) the importance of this metric and suggested that overlayfs could be changed to provide better cold-cache performance as well. Larsson [answered](<https://lwn.net/ml/linux-kernel/CAL7ro1Hc4npP9DQjzuWXJYPTi9H=arLstAJvsBgVKzd8Cx8_tg@mail.gmail.com/>) that, if overlayfs could be modified to address the performance gap, it might be the better solution; he also [raised doubts](<https://lwn.net/ml/linux-kernel/3d4b17795413a696b373553147935bf1560bb8c0.camel@redhat.com/>) as to whether the performance gap could really be closed and whether it made sense to add more complexity to overlayfs. 

The conclusion from this part of the discussion was that some experimentation with overlayfs made sense to see whether it is a viable option or not. Overlayfs maintainer Miklos Szeredi has been mostly absent from the discussion, but did briefly [indicate](<https://lwn.net/ml/linux-kernel/CAJfpeguczp-qOWJgsnKqx6CjCJLV49j1BOWs0Yxv93VUsTZ9AQ@mail.gmail.com/>) that some of the proposed changes might make sense. 

#### The EROFS option

There was another option that came up a number of times in the discussion, though: enhance the EROFS filesystem to include the functionality provided by composefs. This filesystem, which is designed for embedded, read-only operation, already has fs-verity support. EROFS developer Gao Xiang has repeatedly [said](<https://lwn.net/ml/linux-kernel/3ae1205a-b666-3211-e649-ad402c69e724@linux.alibaba.com/>) that the filesystem could be enhanced to implement the features provided by composefs; indeed, much of that functionality is already there as part of the [Nydus](<https://github.com/dragonflyoss/image-service/blob/master/docs/nydus-design.md>) mechanism. Scrivano has [questioned](<https://lwn.net/ml/linux-kernel/878rhvg8ru.fsf@redhat.com/>) this idea, though: 

> Sure composefs is quite simple and you could embed the composefs features in EROFS and let EROFS behave as composefs when provided a similar manifest file. But how is that any better than having a separate implementation that does just one thing well instead of merging different paradigms together? 

Gao has suggested that the composefs developers will, sooner or later, want to add support for storing file data (rather than just the manifest metadata), at which point composefs will start to look more like an ordinary filesystem. At such a point, the argument for separating it from other filesystems would not be so strong. 

A few performance issues in EROFS (for this use case) were identified in the course of the discussion, and various fixes have been implemented. Jingbo Xu has run [a number of benchmarks](<https://lwn.net/ml/linux-kernel/a526afef-a5ce-ceec-d5b7-1da9fab1a18f@linux.alibaba.com/>) to measure the results of patches to both EROFS and overlayfs, but none yet have shown that either of the other two options can outperform composefs. That work is still in an early state, though. 

As might be imagined, this sprawling conversation did not come to any sort of consensus with regard to whether it makes sense to merge composefs or to, instead, put development effort into one of the alternatives. Chances are that no such conclusion will be reached for the next few months. This is just the sort of decision that the upcoming [LSFMM/BPF summit](<https://events.linuxfoundation.org/lsfmm/>) was created to resolve; chances are that there will be an interesting discussion at that venue.
