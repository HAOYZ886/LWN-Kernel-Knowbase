---
title: Sysfs and namespaces
url: https://lwn.net/Articles/295587/
date: "August 26, 2008"
category: Namespaces; Sysfs
author: "By Jonathan Corbet August 26, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 26, 2008

Support for network namespaces - allowing different groups of processes to have a different view of the system's network interfaces, routes, firewall rules, etc. - is nearing completion in recent kernels. A look at `net/Kconfig` turns up something interesting, though: network namespaces can only be enabled in kernels which do not support sysfs - the two are mutually exclusive. Since most system configurations need sysfs to work properly, this limitation has made it harder than it would otherwise be to use, or even test, the network namespace feature. 

The problem is simple: the network subsystem creates a sysfs directory for each network interface on the system. For example, `eth0` is represented by `/sys/class/net/eth0`; therein one can find out just about anything about how `eth0` is configured, query its packet statistics, and more. But, when network namespaces are in use, one group of processes may have a different `eth0` than another, so they cannot share a globally-accessible sysfs tree. One solution might be to add the network namespace as an explicit level in the sysfs tree, but that would break user-space tools and fails to properly isolate namespaces from each other. The real solution is to build namespace awareness more deeply into sysfs. 

Eric Biederman has been working on a set of sysfs namespace patches for the last year or so; those patches now appear to be getting close to ready for inclusion into the mainline. Network namespaces will be the first user of this feature, but it has been written in a way that makes it possible for any system namespace to provide differing views of parts of the sysfs hierarchy. 

The core concept is that of a "tagged" directory in sysfs. Any sysfs directory can be associated with (at most) one type of tag, where that type identifies which type of namespace controls how that directory is viewed. Thus, for example, `/sys/class/net` would have a tag type identifying the network namespace subsystem as the one which is in control there. The `/sys/kernel/uids` directory, instead, will be managed by the user namespace subsystem. Once a directory is given a tag type, all subdirectories and attribute files inherit the same type. 

Namespace code makes use of tagged sysfs directories by adding an entry to `enum sysfs_tag_type`, defined in `<linux/sysfs.h>`, to identify its specific tag type. The namespace must also create an operations structure: 

```
struct sysfs_tag_type_operations {
    	const void *(*mount_tag)(void);
        };
```

The purpose of the `mount_tag()` method is to return a specific tag (represented by a `void *` pointer) for the `current` process. This tag, normally, will just be a pointer to the structure describing the relevant namespace; for example, network namespaces implement this method as follows: 

```
static const void *net_sysfs_mount_tag(void)
        {
    	return current->nsproxy->net_ns;
        }
```

The tag operations must be registered with sysfs using: 

```
int sysfs_register_tag_type(enum sysfs_tag_type type, 
                                    struct sysfs_tag_type_operations *ops);
```

Thereafter, there are two ways of associating tags with a sysfs hierarchy. One of those is to make a tagged directory directly with: 

```
int sysfs_make_tagged_dir(struct kobject *kobj, 
                                  enum sysfs_tag_type type);
```

The directory associated with `kobj` will have differing contents depending on the value of the tag of the given `type`. The actual tag associated with the contents of this directory will be determined (at creation time) by calling a new function added to the `kobj_type` structure: 

```
const void *(*sysfs_tag)(struct kobject *kobj);
```

The `sysfs_tag()` function is usually a short series of `container_of()` calls which, eventually, locates the appropriate namespace for the given `kobj`. 

An alternative way to attach tags to a directory tree is to associate it directly with the `class` structure. To that end, `struct class` has two new members: 

```
enum sysfs_tag_type tag_type;
        const void *(*sysfs_tag)(struct device *dev);
```

When the class is instantiated, it will have tags of the given `tag_type`; the specific tag for a given class will be found by calling the `sysfs_tag()` function. 

Finally, if a specific tag ceases to be valid (because the associated namespace is destroyed, normally), a call should be made to: 

```
void sysfs_exit_tag(enum sysfs_tag_type type, const void *tag);
```

This call will cause all sysfs directories with the given `tag` to become invisible, and to be deleted when it is safe to do so. 

Adding tagged directory support requires some significant changes to the sysfs code. But the interface has been designed to make it very easy for other subsystems to make use of tagged directories; it's a simple matter of providing functions to return the specific tag values which should be used. At this point, the biggest challenge might be making sense of sysfs when its contents may be different for each observer. But that is a challenge associated with namespaces in general.
