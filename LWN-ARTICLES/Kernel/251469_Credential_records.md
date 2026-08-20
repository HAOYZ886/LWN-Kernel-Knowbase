---
title: Credential records
url: https://lwn.net/Articles/251469/
date: "September 25, 2007"
category: "Credentials; FS-Cache; Security-Security technologies"
author: "By Jonathan Corbet September 25, 2007"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 25, 2007

Every Linux process carries with it a set of credentials which describe its privileges within the system. Credentials include the user ID, group membership, capabilities, security context, and more. These credentials are currently stored in the `task_struct` structure associated with each process; an operation which changes credentials does so by operating directly on the `task_struct` structure. This approach has worked for many years, but it occasionally shows its age. 

In particular, the current scheme makes life hard for kernel code which needs to adopt a different set of credentials for a limited time. In an attempt to remedy that situation, David Howells has posted [a patch](<http://lwn.net/Articles/251224/>) which significantly changes the handling of process credentials. The result is a more complex system, but also a system which is more flexible, and, with luck, more secure. 

The core idea behind this patch is that all process credentials (attributes which describe how a process can operate on other objects) should be pulled out of the task structure into a separate structure of their own. The result is `struct cred`, which holds the effective filesystem user and group IDs, the list of group memberships, the effective capabilities, the process keyrings, a generic pointer for security modules, and some housekeeping information. The result is quite a bit of code churn as every access to the old credential information is changed to look into the new `cred` structure instead. 

That churn is complicated by the fact that quite a bit of the credential information has not really _moved_ to the `cred` structure; instead it is mirrored there. One of the fundamental rules for how `struct cred` works is that the structure can only be changed by the process it describes. So anything in the structure which can be changed by somebody else - capabilities and keyrings, for example - remain in the `task_struct` structure and are copied into the `cred` structure as needed. "As needed," for all practical purposes, means anytime those credentials are to be checked. So most system calls get decorated with this extra bit of code: 

```
result = update_current_cred();
        if (result < 0)
            return result;
```

The next rule says that the `cred` structure can never be altered once it has been attached to a task. Instead, a read-copy-update technique must be used, wherein the `cred` structure is copied, the new copy is changed, then the pointer from the `task_struct` structure is set to the new structure. The old one, which is reference counted, persists while it is in use and is eventually disposed of via RCU. 

There is a whole set of utility functions for dealing with credentials, a few of which are: 

```
struct cred *get_current_cred();
        void put_cred(struct cred *cred);
```

A call to `get_current_cred()` takes a reference to the current process's `cred` structure and returns a pointer to that structure. `put_cred()` releases a reference. 

A change to a credentials structure usually involves a set of calls to: 

```
struct cred *dup_cred(const struct cred *cred);
        void set_current_cred(struct cred *cred);
```

The current credentials can be copied with `dup_cred()`; the duplicate, once modified, can be made current with `set_current_cred()`. A set of new hooks has been added to allow security modules to participate in the duplication and setting of credentials. 

So far, this infrastructure may seem like a bunch of extra work with the gain yet to be explained. The direction that David is going with this change can be seen with this new function: 

```
struct cred *get_kernel_cred(const char *service,
    			         struct task_struct *daemon);
```

The purpose of this function is to create a new credentials structure with the requisite privileges for the given `service`. The `daemon` pointer indicates a current process which should be used as the source for the new credentials - essentially, the new `cred` structure will enable its holder to act as if it were the `daemon` process. The current security module gets a chance to change how those credentials are set up; in fact, the interpretation of the "service" string is only done in security modules. In the absence of a security module, `get_kernel_cred()` will just duplicate the credentials held by `daemon`. 

This capability is used in a new version of David's venerable FS-Cache (formerly [cachefs](<http://lwn.net/Articles/100321/>)) patch set. FS-Cache implements a local cache for network-based filesystems; the locally-stored cache will, naturally, have all of the security concerns as the remote filesystem. There is a daemon which does a certain amount of the cache management work, but other accesses to the cache are performed by FS-Cache code running in the context of a process which is working with files on the remote filesystem. Using the above function, the FS-Cache code is able to empower any process to work with the privileges of the daemon process for just as long as is needed to get the filesystem work done. 

The end result is that security policies can be carried further into the kernel than before. In the FS-Cache case, kernel code doing caching work always operates under the effective capabilities of the cache management daemon. So any protections, SELinux policies, etc. which apply to the daemon will also apply when FS-Cache work is being done in a different context. This should result in a more secure system overall. 

The credential work is still in a relatively early state with a fair amount of work yet to be done. It will be quite a big patch by the time the required changes are made throughout the kernel. So this is not a 2.6.24 candidate. The work is progressing, though, so it will likely be knocking on the mainline door at some point.
