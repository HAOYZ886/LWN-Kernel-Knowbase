---
title: A new approach to user namespaces
url: https://lwn.net/Articles/491310/
date: "April 10, 2012"
category: "Containers; Namespaces-User namespaces"
author: "By Jonathan Corbet April 10, 2012"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 10, 2012

"Containers" can be thought of as a lightweight form of virtualization. Virtualized guests appear to be running on their own dedicated hardware; containers, instead, run on the host's kernel, but in an environment where they appear to have that kernel to themselves. The result is more efficient; it is typically possible to run quite a few more containerized guests than virtualized guests on a given system. The cost, from the user's point of view, is flexibility; since virtualized guests can run their own kernel, they can run any operating system while containerized guests are stuck with what the host is running. 

There is another cost to containers that is not necessarily visible to users: their implementation in the kernel is, in many ways, far more complex. In a system supporting containers, any globally-visible resource must be wrapped in a namespace layer that provides each container with its own view. There are many such resources on a Linux system: process IDs, filesystems, and network interfaces, for example. Even the system name and time can differ from one container to the next. As a result, despite years of effort, Linux still lacks proper container support, while virtualization has been a solid feature for a long time. 

Work continues on the Linux containers implementation; the latest piece is [a new set of user namespace patches](<https://lwn.net/Articles/491236/>) from Eric Biederman. The "user namespace" can be thought of as the encapsulation of user/group IDs and associated privilege; it allows the owner of a container to run as the root user within that container while isolating the rest of the system from the in-container users. The implementation of a proper user namespace has always been a hard problem for a number of reasons. Kernel code has been written with the understanding that a given process has one, globally-recognized user ID; what happens when processes can have multiple user IDs in different contexts? It is not hard to imagine developers making mistakes with user IDs - mistakes leading to serious security holes in the host system. Fears of this kind of mistake, along with the general difficulty of the problem, have kept a full user namespace out of the kernel so far. 

Eric's patch takes a bit of a different approach to the problem in the hope of making user namespaces invisible to most kernel developers while catching errors when they are made. To that end, the patch set creates a new type to represent "kernel UIDs" - `kuid_t`; there is also a `kgid_t` type. The kernel UID is meant to describe a process's identity on the base system, regardless of any user IDs it may adopt within a container; it is the value used for most privilege checks. A process's kernel IDs may (or may not) be equal to the IDs it sees in user space; there is no way for that process to know. The kernel IDs exist solely to identify a process's credentials within the kernel itself. 

In the patch, `kuid_t` is a typedef for a single-field structure; its main reason for existence is to be type-incompatible with the simple integer user and group IDs used throughout the kernel. Container-specific IDs, instead, retain the old integer (`uid_t` and `gid_t`) types. As a result, any attempt to mix kernel IDs with per-container IDs should yield an error from the compiler. That should eliminate a whole class of potential errors from kernel code that deals with user and group ID values. 

The `kuid_t` type, being an opaque cookie to in-kernel users, needs a set of helper functions. Comparisons can be done with functions like `uid_eq()`, for example; similar functions exist for most arithmetic tests. For many purposes, that's all that is needed. Regardless of its namespace, a process's credentials are stored in the global `kuid_t` space, so most ID tests just do the right thing. 

There are times, though, where it is necessary to convert between a kernel ID and a user or group ID as seen in a specific namespace. Perhaps the simplest example is a system call like `getuid()`; it should return the namespace-specific ID, not the kernel ID. There is a set of functions provided to perform these conversions: 

```
kuid_t make_kuid(struct user_namespace *from, uid_t uid);
        uid_t from_kuid(struct user_namespace *to, kuid_t kuid);
        uid_t from_kuid_munged(struct user_namespace *to, kuid_t kuid);
        bool kuid_has_mapping(struct user_namespace *ns, kuid_t uid);
```

(A similar set of functions exists for group IDs, of course.) Conversion between a kernel ID and the namespace-specific equivalent requires the use of a mapping stored within the namespace, so the namespace of interest must be supplied when calling these functions. For code that is executing in process context, a call to `current_user_ns()` is sufficient to get that namespace pointer. `make_kuid()` will turn a namespace-specific UID into a kernel ID, while `from_kuid()` maps a kernel ID into a specific namespace. If no mapping exists for the given kernel ID, `from_kuid()` will return `-1`; for cases where a valid ID must be returned, a call to `from_kuid_munged()` will return a special "unknown user" value instead. To simply test whether it is possible to map a specific kernel ID into a namespace, `kuid_has_mapping()` is available. 

Actually setting up the mapping is a task that must be performed by the administrator, who will likely set aside a range of IDs for use within a specific container. The patch series adds a couple of files under `/proc/_pid_ /` for this purpose; setting up the mapping is just a matter of writing one or more tuples of three numbers to `/proc/_pid_ /uid_map`: 

```
first-ns-id  first-target-id  count
```

The `first-ns-id` is the first valid user ID in the given process's namespace. It is likely to be zero - the root ID is valid and harmless within user namespaces. That first ID will be mapped to `first-target-id` as it is defined in the parent namespace, and `count` is the number of consecutive IDs that exist in the mapping. If multiple levels of namespaces are involved, there may be multiple layers of mapping, but those layers are flattened by the mapping code, which only remembers the mapping directly to and from kernel IDs. 

Establishing mappings clearly needs to be a privileged operation, so the process performing this operation must have the `CAP_SETUID` capability available. A process running as root within its own namespace may well have that capability, even though it has no access outside of its namespace. So processes in a user namespace can set up their own sub-namespaces with arbitrary mappings - but those mappings can only access user and group IDs that exist in the parent namespace. As an additional restriction, the namespace ID mapping can only be set once; after it has been established for a given namespace, it is immutable thereafter. 

These mapping functions find their uses in a number of places in the core kernel. Any system call that deals in user or group ID numbers must include the conversions to and from the kernel ID space. The ext* family of filesystems allows the specification of a UID that can use reserved blocks, so the filesystem code must make sure it's working with kernel IDs when it does its comparisons. So the patch is, like much of the namespace work, mildly intrusive, but Eric has clearly worked to minimize its impact. In particular, he has taken care to ensure that the runtime overhead of the ID-mapping code is zero if user namespaces are not configured into the kernel, and as close as possible to zero when the feature is used. 

The user namespace feature, he [says](<https://lwn.net/Articles/491360/>), now has a number of nice features to offer: 

With my version of user namespaces you no longer have to worry about the container root writing to files in /proc or /sys and changing the behavior of the system. Nor do you have to worry about messages passed across unix domain sockets to d-bus having a trusted uid and being allowed to do something nasty. 

It allows for applications with no capabilities to use multiple uids and to implement privilege separation. I certainly see user namespaces like this as having the potential to make linux systems more secure. 

As of this writing, there have been few comments from reviewers, so it is not yet clear whether other developers agree with this assessment or not. The nature of the namespace work means that it can be difficult to get it accepted into the mainline, where developers tend to be concerned about the overhead and relatively uninterested in the benefits. But, with years of persistence, much of this work has gotten in anyway; chances are that user namespaces, in some form, will eventually find their way in as well.
