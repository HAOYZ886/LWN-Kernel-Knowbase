---
title: Module parameters in sysfs
url: https://lwn.net/Articles/85443/
date: "May 18, 2004"
category: "Modules-Parameters in sysfs; module param"
author: 
---

In the 2.6 kernel, parameters to loadable modules are set up with the `module_param()` macro: 

```
module_param(name, type, perm);
```

The `perm` parameter was set aside for the sysfs representation of this parameter but has, until now, been unused; almost every declared parameter simply sets it to zero in the 2.6.6 kernel. [A new patch](<https://lwn.net/Articles/85654/>) has been posted, however, which makes module parameters in sysfs a reality. 

This patch creates a new `/sys/module` directory; a subdirectory will be created for each module loaded into the system. For unloadable modules, a read-only parameter (called `refcnt`) will be set up which contains the module's current reference count. There will also be attributes for every module parameter whose `perm` value is not zero; that value will, as expected, set the permissions mask for that parameter. 

If the permissions mask allows, module parameters will be writable. In theory, this will give module authors an easy way to export administrator-tweakable knobs to user space. It is worth noting, however, that there is no mechanism for notifying a module that one of its parameters has been changed. Module authors, thus, will have to be careful to ensure that their modules will properly detect and respond to changes to parameters at any time before exporting those parameters in a writable mode. Even so, this patch represents the tying-up of yet another 2.6 loose end.
