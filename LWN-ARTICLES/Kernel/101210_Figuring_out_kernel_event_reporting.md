---
title: Figuring out kernel event reporting
url: https://lwn.net/Articles/101210/
date: "September 8, 2004"
category: "Events reporting; Sysfs-Events layer; kobject hotplug-Use in kernel events layer; send kevent"
author: 
---

Robert Love's kernel event notification patch was covered here [last July](<http://lwn.net/Articles/94605/>). This patch enables the reporting of events to interested user-space software, which can then communicate with the user and generally respond to the events. As the Linux desktop projects become more capable and all-encompassing, they need to know more about what is going on with the system; the events layer is meant to be the mechanism which makes that information available. 

Robert has recently posted [a new version of the patch](<http://lwn.net/Articles/100180/>) which changes the proposed interface significantly. It looks, however, like the patch will change yet again. As it turns out, there is still a fair amount of uncertainty about how best to represent and report kernel events. 

The initial version of the patch required four pieces of information for each event: the type (a general class, like "hotplug"), the object generating the event, the signal (saying what is happening), and an explanatory string. The new version eliminates the descriptive string, and turns the object into a proper kobject, which will be communicated to user space as its location in sysfs. This interface is simpler, and it solves the problem of how to generate predictable and consistent object names, but there are still questions on how events should really be represented. 

The easier part of the discussion has to do with the "type" parameter, which allows user-space applications to filter out events which will not be of interest. Kernel-generated events are expected to be relatively rare, however, so there will be little cost in simply receiving all of them and ignoring the uninteresting ones. So the type value associated with events may go away. 

The more interesting question has to do with the representation of the "signal" parameter. That signal is currently a verb, describing something which has happened with the object of interest. If the object is a CPU, the signal might be "`overheating`". An alternative implementation would be to replace the signal with an attribute of the object; for a processor event, the `temperature` attribute would be passed. User space would then read the value of that attribute in sysfs to figure out what is really going on. This approach would force a structure onto the signal names, and would point user space to where it needs to go to learn more about what is going on. On the other hand, there may not always be attributes available to describe a given event, and the approach could be seen as overly restrictive. 

Meanwhile, Greg Kroah-Hartman [pointed out](<https://lwn.net/Articles/101211/>) that the simplified `send_kevent()` interface strongly resembles another, existing kernel interface: 

```
int send_kevent(struct kobject *kobj, const char *signal);
        void kobject_hotplug(const char *action, struct kobject *kobj);
```

Given that `kobject_hotplug()` is also an event reporting mechanism, why not unify the two? The big difference, at this point, would seem to be that `send_kevent()` uses the netlink interface to communicate with user space, while the hotplug code runs `/sbin/hotplug` and passes the relevant information via the environment. Perhaps the best thing to do, says Greg, is to have the hotplug code also send a copy of its events via netlink, and use it for everything? 

The idea of sending the same events out by way of two different transports does not appeal to many developers, however; it seems better to decide which is best and go with it. The netlink transport is strongly favored by the desktop crowd, which dislikes the unpredictable delays and ordering associated with event handling via `/sbin/hotplug`. On the other hand, netlink is not available early in the boot process, but it is important to be able to handle hotplug events then. 

In the end, the hybrid approach may persist for some time. A future system might use `/sbin/hotplug` at boot time, then turn it off once everything is up and running. The one sure conclusion is that this is an area in need of further thought and experimentation.
