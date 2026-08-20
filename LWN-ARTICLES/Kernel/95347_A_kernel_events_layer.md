---
title: A kernel events layer
url: https://lwn.net/Articles/95347/
date: "July 27, 2004"
category: Events reporting; send kevent
author: 
---

As Linux desktop implementations become more sophisticated, they increasingly need to know about what is going on in the kernel. The desktop code would like to be able to respond properly to events like "disc inserted," "disk full," "processor overheating," "printer on fire," and so on. So far, much of this functionality has been implemented by polling devices and `/proc` files and looking for changes. That solution is, to say the least, inelegant. 

As a way of improving things, Robert Love has posted [a patch](<http://lwn.net/Articles/94913/>) (since [updated](<https://lwn.net/Articles/95296/>)) adding a kernel event notification. This patch, initially by Arjan van de Ven, uses the netlink mechanism to broadcast events out to interested user-space processes. The intent is for the events to be further redistributed using [D-BUS](<http://dbus.freedesktop.org/>), but other uses are possible. 

Within the kernel, events are created with a call to `send_kevent()`: 

```
int send_kevent(enum kevent type, 
                        const char *object,
                        const char *signal,
                        const char *fmt, ...);
```

The `type` argument gives the broad class of the event; current options are `KEVENT_GENERAL`, `KEVENT_STORAGE`, `KEVENT_POWER`, `KEVENT_FS`, and `KEVENT_HOTPLUG`. The `object` is a unique string giving the source of the event; it is derived from the location of the source file in the kernel tree. The `signal` says what is actually happening, and the rest of the arguments are a `printk()`-style format string and arguments giving further information. The patch only adds one set of calls, for noting CPU temperature transitions; they look like: 

```
send_kevent(KEVENT_GENERAL,
                    "/org/kernel/arch/kernel/cpu/temperature", "high",
                    "Cpu: %d\n", cpu);
```

The patch as a whole is not particularly controversial, but there are some concerns about the "object" namespace. Some developers would like to see the mechanism more closely tied into the device model, so that the object as represented here is related to an object in the sysfs hierarchy. Some have asked whether this mechanism should replace the current hotplug interface; that is not the intent, however. There has also been a call for some wrappers to ensure that, for example, device drivers all generate the same sort of event for the same kind of situation. 

This is all detail work; chances are that the event code will find its way into the mainline in some form. Then there is the little issue of making the desktop actually respond to these events in a useful way. But that, of course, is just a user-space problem.
