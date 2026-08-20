---
title: Divorcing namespaces from processes
url: https://lwn.net/Articles/377109/
date: "March 3, 2010"
category: "Containers; Namespaces; Virtualization-Containers"
author: "By Jonathan Corbet March 3, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 3, 2010

For the last few years, the development community interested in implementing containers has been working to add a variety of namespaces to the kernel. Each namespace wraps around a specific global kernel resource (such as the network environment, the list of running processes, or the filesystem tree), allowing different containers to have different views of that resource. Namespaces are tightly tied to process trees; they are created with new processes through the use of special flags to the `clone()` system call. Once created, a namespace is only visible to the newly-created process and any children thereof, and it only lives as long as those processes do. That works for many situations, but there are others where it would be nice to have longer-lived namespaces which are more readily accessible. 

To that end, Eric Biederman has [proposed](<http://lwn.net/Articles/376580/>) the creation of a pair of new system calls. The first is the rather tersely named `nsfd()`: 

```
int nsfd(pid_t pid, unsigned long nstype);
```

This system call will find the namespace of the given `nstype` which is in effect for the process identified by `pid`; the return value will be a file descriptor which identifies - and holds a reference to - that namespace. The calling process must be able to use `ptrace()` on `pid` for the call to succeed; in the current patch, only network namespaces are supported. 

Simply holding the file descriptor open will cause the target namespace to continue to exist, even if all processes within it exit. The namespace can be made more visible by creating a bind mount on top of it with a command like: 

```
mount --bind /proc/self/fd/_N_ /somewhere
```

The other piece of the puzzle is `setns()`: 

```
int setns(unsigned long nstype, int fd);
```

This system call will make the namespace indicated by `fd` into the current namespace for the calling process. This solves the problem of being able to enter another container's namespace without the somewhat strange semantics of the once-proposed [`hijack()` system call](<http://lwn.net/Articles/260172/>). 

These new system calls are in an early, proof-of-concept stage, so they are likely to evolve considerably between now and the targeted 2.6.35 merge.
