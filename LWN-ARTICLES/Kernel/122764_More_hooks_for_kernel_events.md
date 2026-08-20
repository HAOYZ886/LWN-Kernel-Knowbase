---
title: More hooks for kernel events
url: https://lwn.net/Articles/122764/
date: "February 9, 2005"
category: Auditing; Events reporting; Security
author: 
---

The kernel has, for a while now, been accumulating hooks for informing user space when things happen. Some of the current mechanisms include: 

  * The hotplug mechanism, which invokes a user-space program (`/sbin/hotplug` by default) when kobjects are registered or unregistered (generally in response to the addition or removal of hardware on the system). 

  * The Linux security module (LSM) hooks, which enable a loadable module to respond to (and possibly veto) dozens of actions by user-space processes. The LSM mechanism is used by, among other things, SELinux and the realtime LSM module. 

  * The [lightweight audit framework](<https://lwn.net/Articles/79326/>) uses a netlink socket to pass information on kernel events to user space, with the idea that these events will be logged somewhere. 

  * The [kernel events mechanism](<https://lwn.net/Articles/107314/>), which also uses netlink, is a simple scheme for notifying user space of events which might be of interest to the user(s). 

One might think that, at this point, the kernel is sufficiently well instrumented that more hooks would be unnecessary. But more are on the way. 

One of those is the [relay fork module](<https://lwn.net/Articles/122446/>), proposed by Guillaume Thouvenin. Its sole purpose is to inform interested user-space processes when a process forks; the intended user is the [enhanced Linux system accounting project](<http://elsa.sourceforge.net/>). Rather than use one of the existing mechanisms for conveying information to user space, the relay fork patch works by sending a signal to the interested process(es) whenever a fork occurs. 

The patch works by adding a new sysfs directory (`/sys/relayfork`) with a couple of control attributes. The attribute `signal` controls which signal is sent; by default, signal 33 (which is in the realtime signal range on most architectures) is used. The other attribute (`processes`) contains a list of the processes receiving these signals. Registering a process for receipt of "relay fork" signals is simply a matter of writing its process ID to the `processes` attribute. 

This patch may eventually go in, but probably not with the signal mechanism. Guillaume was encouraged to use the kernel events mechanism instead, and he has agreed that it is a workable solution. 

Meanwhile, the [vSecurity project](<https://lwn.net/Articles/121298/>) is working to put together a number of hardening technologies in a form suitable for merging into the mainline. To that end, a couple of new LSM hooks have been proposed. [This one](<https://lwn.net/Articles/122465/>) adds a hook for invocations of the `chroot()` call, which, interestingly, has no such hook now. The purpose is not so much to control the use of `chroot()` as to note that it has happened and take steps, in other security hooks, to ensure that the process does not break out of its restricted subtree. 

The other patch [adds a hook to `chmod()`](<https://lwn.net/Articles/122574/>). This one is unlikely to be merged, since a separate hook, which is called for inode attribute changes, already exists. The vSecurity hacker (Lorenzo Hernandez Garcia-Hierro) has indicated that he has other hooks he wishes to place, but those have not yet been posted for review.
