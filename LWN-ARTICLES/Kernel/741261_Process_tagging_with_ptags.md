---
title: Process tagging with ptags
url: https://lwn.net/Articles/741261/
date: "December 13, 2017"
category: "Containers; Security-Security modules"
author: "By Jonathan Corbet December 13, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
December 13, 2017

For various reasons related to accounting and security, there is recurring interest in having the kernel identify the container that holds any given process. Attempts to implement that functionality tend to run into the same roadblock, though: the kernel has no concept of what a "container" is, and there is seemingly little desire to change that state of affairs. A solution to this problem may exist in the form of a neglected patch called "ptags", which enables the attachment of arbitrary tags to processes. 

Given that containers are at the receiving end of a lot of attention currently, it is natural to wonder why the kernel refuses to recognize them. The kernel does provide the features needed to implement containers: namespaces for isolation, control groups for resource management, seccomp and security modules to implement security policies, etc. But there is little agreement over what actually constitutes a container, and there is still a lot of experimentation going on with interesting new ways of implementing the container concept. When, as part of the recent [discussion on container IDs for auditing](<https://lwn.net/Articles/740621/>), it was suggested that use of namespaces identified a container, Casey Schaufler [responded](<https://lwn.net/Articles/741263/>): 

You might think so, but I am assured that you can have a container without using namespaces. Intel's "Clear Containers", which use virtualization technology, are one example. I have considered creating "Smack Containers" using mandatory access control technology, more to press the point that "containers" is a marketing concept, not technology. 

An attempt to codify such a diverse and rapidly evolving concept (be it a "marketing concept" or not) into a kernel API is likely to end in tears. It would have a strong chance of either stifling ongoing container development or just proving to not be useful with next year's idea of what a container should be. So there is indeed a good case to be made for not recognizing the "container" concept inside the kernel. 

That position may be entirely logical, but it doesn't make the use cases for identifying containers and associating processes with them go away. More than once, Schaufler has suggested that a module called "ptags" is a better solution to this problem, so your editor decided to go take a look. 

Ptags is a proposed security module that was [posted](<https://lwn.net/Articles/703064/>) to the LSM list a few times by José Bollo in late 2016. It received little attention at the time and appears to have disappeared into that place where unloved kernel patches go. There is [a GitLab repository](<https://gitlab.com/jobol/ptags>) for the project, but it has not seen any commits since early February. Ptags has clearly stalled; perhaps what the project needs is some wider attention and more feedback. 

As one might expect, ptags enables the addition of tags to processes. Those tags can be seen and manipulated through a new `/proc` file: `/proc/_PID_ /attr/ptags`. Individual threads of a process can have their own tags in `/proc/_PID_ /tasks/_TID_ /attr/ptags`. Tags are UTF-8 strings (up to 4000 bytes in length, which may be a bit excessive), optionally associated with a string value (32,700 bytes or less — ditto). There are some limitations on control characters, but just about anything goes, so valid tags would include: 

```
IS_EVIL
        CONTAINER_ID=ae883c
        कंटेनर=विपणन
```

The colon character has a special meaning: it is used as a sort of namespace separator. So, for example, if a system were running the Ultimate Marketing Container Manager (UMCM), it might tag processes with their container IDs using something like: 

```
UMCM:CONTAINER_ID=foo
```

If a process is allowed to change some other process's tags (more on that below), such changes are effected by writing to the appropriate `ptags` file. Preceding a tag with "`+`" adds that tag to a process, while "`-`" removes it. Normally a process's tags will be stripped if it calls `execve()`, but that behavior can be changed by prepending "`@`" (the "keep flag") to the tag name. Tags are copied when a process calls `clone()` or `fork()`, though. There is a simple glob mechanism for deleting tags or changing keep flags in bulk. 

By default, unprivileged processes cannot change tags — neither their own nor another process's. Permissions to change tags with a specific namespace prefix can be delegated using the tag system itself. If the administrator wanted the UMCM process to be able to control tags starting with `UMCM:` on other processes, the UMCM process would be given one or more of these tags: 

```
ptags:UMCM:add
        ptags:UMCM:sub
        ptags:UMCM:set
        ptags:UMCM:others
```

The first tag allows the UMCM process to add tags starting with `UMCM:` to itself. The "`sub`" tag allows removing those tags from itself, and "`set`" allows changing existing tags. The "`others`" tag is different, in that it causes any other permissions on the `UMCM:` namespace to apply globally. If a process's tags include both `ptags:UMCM:add` and `ptags:UMCM:others`, it can add tags in the `UMCM:` namespace to any other process in the system. That permission does also require that the process in question can write to the target process's `ptags` file, which may be restricted by access permissions or another security module. 

Other than the special `ptags:` tags, nothing in the kernel uses or cares about process tags in any way. They are maintained as a service for user space, making it easy to associate information with processes in a way that those processes cannot change. It would seem that this sort of mechanism would work well for the container use case; a container manager could tag processes in a way that matches its particular scheme. Meanwhile, the kernel need not know anything about any particular conception of what a container is. 

One drawback to this scheme, beyond the fact that it's not in the mainline and doesn't appear to be headed that way is that, [according to Schaufler](<https://lwn.net/Articles/741288/>): ""PTAGS unfortunately needs module stacking, but how hard could that be?"" The answer to that question would be "[fairly hard](<https://lwn.net/Articles/635771/>)", but there is another question that is worth asking: does the ptags mechanism need to be a security module at all? The usual point of security modules is to restrict access to system resources in some way, but ptags doesn't do that. 

If the ptags approach looks like the right solution to the container-ID problem, it might be worth implementing it as a core kernel feature. Processes have a long list of attributes in a Linux system; the tags would just be more of the same. That would ensure that tags would be available on the systems that need them, eliminate the stacking problem and, in general, reduce the potential for unfortunate interactions with other security modules. "Container" might not be appropriate as a core-kernel concept, but "process tags" might be. 

But that, of course, would require somebody to either push the existing module forward or implement a similar scheme in another way. But, as Schaufler asked, how hard can that be? As the pressure to solve the container-ID problem continues to grow, some developer may well be motivated to give this approach a try.
