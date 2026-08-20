---
title: Retrieving kernel attributes
url: https://lwn.net/Articles/897420/
date: "June 13, 2022"
category: "Filesystems-Extended attributes"
author: "By Jake Edge June 13, 2022 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
June 13, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

At the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Amir Goldstein and Miklos Szeredi led a discussion on a new interface for extracting information from kernel objects using the filesystem extended-attributes (xattr) interface. Since Szeredi was not present in Palm Springs, he co-led the session virtually over Zoom audio, which was the only filesystem session with a virtual leader at LSFMM this year. Szeredi's [proposal](<https://lwn.net/ml/linux-kernel/YnEeuw6fd1A8usjj@miu.piliscsaba.redhat.com/>) for an interface of that sort had been posted just the day before the session. 

Goldstein started things off by noting that there are several use cases where there is a need for a new API to obtain bits of information from the kernel, so it seems like a good idea to create a common API that can meet those needs. Szeredi proposed the `getfattr` mechanism, which builds on the xattr interface; Goldstein said that he was happy with the idea, as were Szeredi and Dave Chinner, who suggested the idea a year or so ago. In addition, other than an [objection to binary data](<https://lwn.net/ml/linux-kernel/YnFB/ct2Q/yYBnm8@kroah.com/>), Greg Kroah-Hartman was "not unhappy" with the idea. 

Szeredi took over to describe the proposal in more detail. The intent is to be able to get attributes from some kernel objects; those could be mounts or inodes, but processes or other objects are possible as well. There are several existing interfaces for getting this kind of information, but each has a different way to access the attributes, so it would be nice to have a unified interface, he said. 

The xattr API was repurposed for his proposal. It uses a different namespace for the new attributes, however, in order to ensure that legacy code will not break due to unexpected new attributes. For example, [`listxattr()`](<https://man7.org/linux/man-pages/man2/listxattr.2.html>) would not return attributes from the new namespace. One objection to the interface is that it is not efficient enough if there is a need to retrieve multiple attributes. Szeredi said that would need to be tested to see if it is truly a problem, but if so, the API could be extended with a bulk-retrieval mechanism. 

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2022/lsfmm-goldstein2-sm.png) ](<https://lwn.net/Articles/897578/>)

Goldstein said that the same interface could be used for a "`setfattr`" tool that could set attributes; he wondered if there were any objections to the general idea. David Howells said that he had some "potential objections" that are likely surmountable: for example, `getfattr` does not have the right security checks. It should either have no security checks or ones like the [`statfs()`](<https://man7.org/linux/man-pages/man2/statfs.2.html>) system call has. The checks required could be based on the namespace being queried, so that Linux security module (LSM) checks could be accommodated as needed. 

Howells would also rather see the information be returned as binary data, rather than strings, especially for things that need to be retrieved quickly. He has gotten messages from some developers who liked his [`fsinfo()` proposal](<https://lwn.net/Articles/829212/>) because it returned data in binary form, so there was no need to parse it. Goldstein said that others want to be able to use `getfattr` in shell scripts, however. 

The idea is to have a simple and flexible generic interface, he said. If there is a need for higher performance, then once that has been demonstrated, a different interface can be added. Howells said that there is a need for systemd to be able to read a list of thousands of mounts; it will need higher performance. But Goldstein said that most systems do not have thousands of mounts; another interface that is less simple and generic can be added for those kinds of use cases. 

Ted Ts'o said that the interface being proposed is not for reading lists of thousands of mounts, but is, instead, for getting information like: "what is the mount point for that particular file or directory?" Thousands of mount points are a reality on production systems at Google, he said, but this interface is not meant for that case. In his mind, `getfattr` is the non-controversial part; it is the `setfattr` piece that has not been specified which requires a lot more consideration. There are questions of which attributes can be set, what the permissions required are, how the interface can be introspected, and so on. If the setting interface is not done right, he said that Luis Chamberlain would eventually have to give a talk to complain that it is just as [bad as `ioctl()` is](<https://lwn.net/Articles/897202/>). If `getfattr` is the camel's nose under the tent for an unseen set interface, that worries Ts'o. 

Christian Brauner said that the systemd developers should get an opportunity to weigh in on the proposal. There are longstanding bugs and serious performance issues that the tool has experienced when gathering mount properties on thousands of mounts on production workloads. That is part of what is driving the `fsinfo()`-style of interface. Brauner thinks it is important to address those problems in any kind of proposal of this sort. 

Howells said that some kind of "get mountlist" call might be sufficient to solve the specific problems that systemd is experiencing. Goldstein said that it was not necessary for this proposal to solve all of the problems, however. The problem for systemd is that a single change to the mounted filesystems requires it to rescan the mount tree because it does not get notified of what the change is, only that something has changed, Szeredi said. If the notification could somehow be improved, that might solve systemd's problem. 

Goldstein said that the proposal email showed a static hierarchy of attributes but that the hierarchy can be extended flexibly such that each filesystem type could have its own namespace. The CIFS filesystem already does that for both getting and setting attributes in a `cifs.*` namespace. Brauner asked what the new system call underlying `getfattr` looked like. Goldstein replied that it was simply using [`getxattr()`](<https://man7.org/linux/man-pages/man2/getxattr.2.html>). The difference is in the interpretation of the namespaces that are included in the path name argument. 

Ts'o said that it did not really make sense for ext4 to switch to this xattr-based mechanism, since it already has a way for programs to retrieve ext4-specific information via sysfs. That code must be maintained for backward compatibility, so adding more code to support the xattr-based mechanism is not attractive. Trying to force all filesystems and applications to use the proposed interface for filesystem-dependent information is probably a bad idea, he said. Any filesystem that wants to use it, should go ahead and do so, however. He just does not see any real value for ext4. 

Brauner suggested making it a different system call, even if it is actually using the same `getxattr()` code underneath. The current expectation for xattrs is that they are stored on disk associated with a file, which is not the case for "fattrs". Goldstein agreed that it probably makes sense to do so. Two other things to consider are adding a `getxattrat()` system call and, perhaps, a way to get multiple xattrs in a single call, he said. XFS has an `ioctl()` command for getting multiple xattrs, which could perhaps be generalized. With that, the session ran out of time, but it seems that the xattr-based approach will continue to be pushed forward.
