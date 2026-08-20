---
title: Namespace file descriptors
url: https://lwn.net/Articles/407495/
date: "September 29, 2010"
category: "Containers; Namespaces; Virtualization-Containers"
author: "By Jake Edge September 29, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
September 29, 2010

Giving different groups of processes their own view of global kernel resources--network environments and filesystem trees for example--is one of the goals of the kernel container developers. These views, or namespaces, are created as part of a `clone()` with one of the `CLONE_NEW*` flags and are only visible to the new process and its children. Eric Biederman has [proposed](<http://lwn.net/Articles/406684/>) a mechanism that would allow other processes, outside of the namespace-creator's descendants, to see and access those namespaces. 

When we [looked](<http://lwn.net/Articles/377109/>) at an earlier version back in March, Biederman had proposed two new system calls, `nsfd()` and `setns()`. Since that time, he has eliminated the `nsfd()` call by adding a new `/proc/<pid>/ns` directory with files that can be opened to provide a file descriptor for the different kinds of namespaces. That removes the need for a dedicated system call to find and return an fd to a namespace. 

Currently, there must be a process running in a namespace to keep it around, but there are use cases where it is rather cumbersome to have a dedicated process for keeping the namespace alive. With the new patches, doing a bind mount of the proc file for a namespace: 

```
mount --bind /proc/self/ns/net /some/path
```

for example, will keep the namespace alive until it is unmounted. 

The `setns()` call is unchanged from the earlier proposal: 

```
int setns(unsigned int nstype, int nsfd);
```

It will set the namespace of the process to that indicated by the file descriptor `nsfd`, which should be a reference to an open namespace `/proc` file. `nstype` is either zero or the name of the namespace type the caller is trying to switch to ("net", "ipc", "uts", and "mnt" are implemented), so the call will fail if the namespace that is referred to by `nsfd` does not correspond. The call will also fail unless the caller has the `CAP_SYS_ADMIN` capability (root privileges, essentially). 

For this round, Biederman has also added something of a convenience function, in the form of the `socketat()` system call: 

```
int socketat(int nsfd, int family, int type, int protocol);
```

The call parallels `socket()`, but takes an `nsfd` parameter for the namespace to create the socket in. As pointed out in the discussion of that [patch](<https://lwn.net/Articles/407613/>), `socketat()` could be implemented using `setns()`: 

```
setns(0, nsfd);
        sock = socket(...);
        setns(0, original_nsfd);
```

Biederman [agrees](<https://lwn.net/Articles/407615/>) that it could be done in user space, but is concerned about race conditions in an implementation of that kind. In addition, unlike for the other namespace types, he has some specific use cases in mind for network namespaces: 

The use case are applications are the handful of networking applications that find that it makes sense to listen to sockets from multiple network namespaces at once. Say a home machine that has a vpn into your office network and the vpn into the office network runs in a different network namespace so you don't have to worry about address conflicts between the two networks, the chance of accidentally bridging between them, and so you can use different dns resolvers for the different networks. 

But he also realized that it might be a somewhat controversial addition. Overall, there has been relatively little discussion of the patchset on linux-kernel, and Biederman said that it had received positive reviews on the containers mailing list. He posted the patches so that other kernel developers could review the ABI additions, and there seem to be no complaints with `setns()` and the `/proc` filesystem additions. 

Changes for the "pid" namespace were not included in these patches as there is some work needed before that namespace can be safely unshared. That work doesn't affect the ABI, though. Once the pid namespace is added in, it seems likely we will see these patches return, perhaps without `socketat()`, sometime soon. Allowing suitably privileged processes to access others' namespaces will be a useful addition, and one that may not be too far off.
