---
title: Some upcoming sysfs enhancements
url: https://lwn.net/Articles/174660/
date: "March 7, 2006"
category: Relayfs; Sysfs
author: 
---

A glance at Greg Kroah-Hartman's [state of the driver core and sysfs message](<http://lwn.net/Articles/174592/>) shows that a number of changes are queued up for future kernel cycles. A couple of those add new features to sysfs, and seem worth a mention. 

Attribute files in sysfs serve as a channel for sharing information between the kernel and user space. As more of the information interface moves to sysfs, an increasing number of user-space programs will be making use of sysfs attributes. Often, these programs will want to respond when the value of a sysfs attribute changes. In current kernels, however, there is no easy way for an application to know when an attribute has changed; the only option is to repeatedly re-read the file and check for new values. 

The current -mm kernels include a patch by Neil Brown which makes it possible to create pollable attributes. With such attributes, user space need only open the attribute of interest pass it to `poll()` with the `POLLERR` and `POLLPRI` events selected. When `poll()` returns, the file can be reopened and reread to obtain the new value. 

Internally, the patch adds a wait queue head to every kobject on the system; that queue is inserted into a poll table in response to a `poll()` call. The sysfs code has no way of knowing, however, when the value of any given sysfs attribute has changed, so the subsystem implementing a pollable attribute must make explicit calls to: 

```
void sysfs_notify(struct kobject *kobj, char *dir, char *attr);
```

Here, `kobj` and `attr` describe the attribute whose value has been changed. The `dir` argument need only be supplied when the given kobject has a special subdirectory (and the attribute is in that directory). This call will cause any polling process to wake up and see that a new value is available. 

With the current code, there is no way to mark attributes which can be polled. Any process which calls `poll()` on an attribute which does not support polling will end up waiting rather longer than the developer intended. 

While sysfs attributes are normally low-bandwidth items - holding generally a single value - the relayfs subsystem (added in 2.6.14) is meant to be a high-bandwidth pipe from the kernel to user space. Relayfs is often used for debugging tasks, such as relaying large amounts of kernel trace data for later analysis. User space gets at that data stream by opening a channel file created in the special-purpose relayfs filesystem. 

As it turns out, relayfs contains a fairly nice internal abstraction for its file operations, making it possible to create entries for relay channels in other filesystems. Paul Mundt recently put together [a patch](<http://lwn.net/Articles/172691/>) taking advantage of this feature to allow kernel code to create relayfs channels in sysfs. The reaction to this capability was positive; indeed, it was seen as a better interface to the relay code than relayfs itself. So Paul's patches have grown into a full reworking of the relay interface, with the separate relayfs filesystem going away entirely. 

Most of the interfaces remain unchanged; in particular, almost the entire kernel API (as described in [the documentation file](<https://lwn.net/Articles/174669/>)) remains as it was before. But now there is a pair of new functions: 

```
int sysfs_create_relay_file(struct kobject *kobj, 
                                    struct relay_attribute *attr);
        void sysfs_remove_relay_file(struct kobject *kobj, 
                                     struct relay_attribute *attr);
```

A simple call to `sysfs_create_relay_file()` will add a relay channel attribute to the given kobject. The `relay_attribute` structure must be filled in with information about the actual channel. On the user-space side, the only change is that the application must look in a different place to find the relay channel. All of the supported operations (`mmap()` in particular) work as before. 

Barring last-minute objections, both of these patches seem likely to be merged for 2.6.17.
