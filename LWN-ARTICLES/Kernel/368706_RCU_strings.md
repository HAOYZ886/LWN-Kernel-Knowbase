---
title: RCU strings
url: https://lwn.net/Articles/368706/
date: "January 5, 2010"
category: "Read-copy-update"
author: "By Jonathan Corbet January 5, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 5, 2010

The sysctl mechanism has seen a lot of work in recent kernel development cycles, resulting in the removal of a lot of code and a reduction in big kernel lock usage. It turns out, though, that this work has also introduced some subtle and rare race conditions into the handling of string data exported to user space. In response, Andi Kleen has put together a new concept called "RCU strings," using the read-copy-update mechanism to eliminate the races without the introduction of new locks on the read path. 

There are a number of strings managed through sysctl. As an example, consider `request_module()`, which is used by kernel code to ask user space to load a module. A call to `request_module()` will result in an invocation of `modprobe`, but nobody wants to wire the name or location of `modprobe` in kernel code. So the sysctl variable `/proc/sys/kernel/modprobe` is used to contain the location of this utility. It will be set to "`/sbin/modprobe`" on almost any Linux system, but an administrator can change it if need be. 

Consider the case of a `request_module()` call which happens at exactly the same time as a change to `/proc/sys/kernel/modprobe` from user space. It is entirely possible that `request_module()` could end up with the path to `modprobe` which has been partially modified. The most likely result is a failed attempt to load the module, but worse things could happen. This situation is well worth avoiding. 

(One should note that these races are not, in general, potential security problems. The changing of sysctl variables is a privileged operation, so it cannot be done from arbitrary user accounts.) 

The read-copy-update mechanism was designed to ensure that data - especially data which is frequently read but rarely modified - remains stable while it is being used. So it seems well suited to the protection of sysctl strings which, likely as not, will never be changed over the lifetime of the system. RCU can be a bit tricky to use, though; the [RCU string type](<http://lwn.net/Articles/368684/>) is designed to make things a bit easier. 

The creation of an RCU string is accomplished through: 

```
#include <linux/rcustring.h>
    
        char *alloc_rcu_string(int size, gfp_t gfp);
```

The `size` parameter should be the maximum size that the string can be - null byte included. 

Following the normal RCU pattern, read access to this string is accomplished by way of a pointer to that string. Atomic readers - those which do not sleep - need only use `rcu_read_lock()` and `rcu_dereference()` to mark their use of the RCU-protected pointer. Any code which might sleep will have to take other measures, since the string could change while the code is not running. In this case, a copy of the string should be made with: 

```
char *access_rcu_string(char **str, int size, gfp_t gfp);
```

Here, `str` is a pointer to the string pointer, and `size` is the size of the originally-allocated string. Using `strlen()` to get `size` would be a serious mistake, since the string could possibly change before the copy is made. The new string is allocated with `kmalloc()`; the given `gfp` flags are used for the allocation. The copied string should be freed with `kfree()` when it is no longer needed. 

Code changing an RCU string should use `alloc_rcu_string()` to allocate a replacement string, copy the data into it, then use `rcu_assign_pointer()` to make the new string visible to the rest of the system. The old string should be passed to `free_rcu_string()`, which will use RCU to free the memory once it is known that no references to that string can still exist. 

String variables tend to be exported through sysctl using `proc_dostring()`. To make life easier, Andi has added a new function, `proc_rcu_string()`, which handles most of the details of exporting an RCU string. It's a simple matter of initializing the appropriate `ctl_table` structure with a `char **` pointer to the string pointer and setting the `proc_handler` entry to `proc_rcu_string()`. The initial value of the string is allowed to be a compile-time constant string; anything else is expected to be an RCU string. 

This code has been through a couple rounds of review and seems likely to be merged in the 2.6.34 development cycle.
