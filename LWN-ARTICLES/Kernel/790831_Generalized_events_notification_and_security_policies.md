---
title: Generalized events notification and security policies
url: https://lwn.net/Articles/790831/
date: "June 11, 2019"
category: Events reporting
author: "By Jonathan Corbet June 11, 2019"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 11, 2019

Interfaces for the reporting of events to user space from the kernel have been a recurring topic on the kernel mailing lists for almost as long as the kernel has existed; LWN [covered one](<https://lwn.net/Articles/95347/>) 15 years ago, for example. Numerous special-purpose event-reporting APIs exist, but there are none that are designed to be a single place to obtain any type of event. David Howells is the latest to attempt to change that situation with [a new notification interface](<https://lwn.net/ml/linux-kernel/155991702981.15579.6007568669839441045.stgit@warthog.procyon.org.uk/>) that, naturally, uses a ring buffer to transfer events to user space without the need to make system calls. The API itself (which hasn't changed greatly since it was [posted in 2018](<https://lwn.net/Articles/760714/>)) is not hugely controversial, but the associated security model has inspired a few heated discussions. 

#### `/dev/watch_queue`

Howells's mechanism is implemented as a special device, likely to be called `/dev/watch_queue`. Applications start by opening that device for write access; they then need to configure the size of the ring buffer. That is done with the `IOC_WATCH_QUEUE_SET_SIZE` `ioctl()` command, passing the desired size in pages. In the current patch set, the size must be a power of two and no greater than 16\. Once that is done, `mmap()` is used to map the ring buffer into the application's address space. 

The ring buffer itself is divided into eight-byte slots. Entries for specific events can occupy more than one slot; the first slot always contains this structure: 

```
struct watch_notification {
    	__u32	type:24;
    	__u32	subtype:8;
    	__u32	info;
        };
```

The `type` and `subtype` fields tell the application what type of event has occurred; the attachment of a USB device would be reported as `WATCH_TYPE_USB_NOTIFY` and `NOTIFY_USB_DEVICE_ADD`. The special `WATCH_TYPE_META` type and `WATCH_META_SKIP_NOTIFICATION` subtype indicate an entry that should simply be skipped (there are a couple of uses for such entries that will be described below). The `info` field contains flags to report ring overruns or events dropped due to a lack of memory, for example; it also contains a subfield describing the number of slots occupied by this particular event. 

The ring buffer itself looks like this: 

```
struct watch_queue_buffer {
    	union {
    	    struct {
    		struct watch_notification watch; /* WATCH_TYPE_META */
    		__u32		head;		/* Ring head index */
    		__u32		tail;		/* Ring tail index */
    		__u32		mask;		/* Ring index mask */
    	    } meta;
    	    struct watch_notification slots[0];
    	};
        };
```

The use of a union implies that the `meta` structure, containing the `head` and `tail` pointers for the ring, overlays the first three slots. The special `watch` structure embedded in there is marked to be skipped, so user-space code will simply pass over the header information without the need to do anything special. 

The buffer is empty if `head` and `tail` are equal. The kernel will insert the next event at the slot pointed to by `head` and increment `head` by the number of slots used by that event. Events will not be split at the end of the ring; if there are not enough slots left to hold the full event, the ring will be padded with skip events and the new event inserted at the beginning. User space should consume events starting at `tail`, and increment `tail` accordingly when each event is dealt with. As is always the case for data structures like this, appropriate memory barriers should be used when working with the ring indexes. 

The application can call `poll()` to wait for events if need be. 

#### Selecting events

The other piece of the puzzle is telling the kernel which events are of interest to begin with. Each subsystem provides its own way of requesting that events be delivered into a specific buffer. The patch set implements a number of event sources: 

  * Events involving keys can be requested with the `KEYCTL_WATCH_KEY` command to the [`keyctl()`](<http://man7.org/linux/man-pages/man2/keyctl.2.html>) system call. 
  * Filesystem mount and unmount events can be had with a call to the new `watch_mount()` system call. 
  * Events on specific filesystems (deemed "superblock events") are requested with the new `watch_sb()` system call. 
  * Yet another new system call, `watch_devices()`, allows for the requesting of events related to hardware. The patch set adds support for events from the block and USB subsystems. 

Finally, by default all events of the requested type(s) will be delivered into the ring buffer. The application might well only be interested in a small subset of those events. To avoid passing data that is not useful, there is a filtering mechanism built around this structure: 

```
struct watch_notification_type_filter {
    	__u32	type;
    	__u32	info_filter;
    	__u32	info_mask;
    	__u32	subtype_filter[8];
        };
```

The application puts the type of the event of interest into `type`. `subtype_filter` is a bitmask that can be used to limit which event subtypes are delivered; the application sets the bit corresponding to each desired subtype. For more complex filtering, the `info_filter` and `info_mask` fields can be used. Any given event will be delivered if: 

```
(event.info & info_mask) == info_filter
```

In other words, `info_mask` indicates which parts of the `info_field` are of interest, and `info_filter` holds the values that should be found in those parts. 

The application can package up as many filters as it needs into this structure: 

```
struct watch_notification_filter {
    	__u32	nr_filters;
    	__u32	__reserved;
    	struct watch_notification_type_filter filters[];
        };
```

The result is then passed to the kernel with the `IOC_WATCH_QUEUE_SET_FILTER` `ioctl()` command. 

A lot more details about the notification mechanism, including the kernel-side API, can be found in the document at the beginning of [this patch](<https://lwn.net/ml/linux-kernel/155991709157.15579.17714751502536525102.stgit@warthog.procyon.org.uk/>). 

#### Security

Naturally, information out of the kernel could be sensitive and should not be given to any process that might request it. In [an earlier (May 28) version](<https://lwn.net/ml/linux-kernel/155905930702.7587.7100265859075976147.stgit@warthog.procyon.org.uk/>) of the patch set, events related to keys would only be delivered if the recipient has "View" access to the key involved. Information on mount events was unrestricted; superblock events were also unrestricted for any filesystem that was actually visible to the calling process. Generic device events were not a part of that patch set; block-subsystem events were supported as a distinct type and were unrestricted. This policy was seen as being overly loose in a number of ways, one of which was surprising to many of the participants in the discussion. 

Consider mount events in particular, and whether process B should be able to see events generated when process A mounts or unmounts a filesystem. One might argue that B should be privileged, or should at least have enough access to watch what A is doing in general. Casey Schaufler, though, [argued the reverse](<https://lwn.net/ml/linux-kernel/50c2ea19-6ae8-1f42-97ef-ba5c95e40475@schaufler-ca.com/>): for B to see an event generated by A, it is A that should have sufficient privilege to send signals to B: 

If process A sends a signal (writes information) to process B the kernel checks that either process A has the same UID as process B or that process A has privilege to override that policy. Process B is passive in this access control decision, while process A is active. In the event delivery case, process A does something (e.g. modifies a keyring) that generates an event, which is then sent to process B's event buffer. Again, A is active and B is passive. Process A must have write access (defined by some policy) to process B's event buffer. 

Any other policy, he said, would [open covert channels](<https://lwn.net/ml/linux-kernel/058f227c-71ab-a6f4-00bf-b8782b3b2956@schaufler-ca.com/>) between the processes and would be difficult to specify and model in general. To others, though, this policy seemed backward and surprising; most others were also less worried about covert channels than Schaufler is. The discussion circled around a few versions of the patch set with no seeming resolution (though Howells did attempt to implement the policy Schaufler was asking for); at one point Andy Lutomirski [called it](<https://lwn.net/ml/linux-kernel/176F8189-3BE9-4B8C-A4D5-8915436338FB@amacapital.net/>) ""a giant design error"". 

One seemingly counterintuitive example perhaps led to a better understanding between the participants, though. SELinux maintainer Stephen Smalley [pointed out](<https://lwn.net/ml/linux-kernel/c82052e5-ca11-67b5-965e-8f828081f31c@tycho.nsa.gov/>) that, if two processes are both able to map a file, they can communicate via that file, so restricting notifications about activity on that file does not increase security. Schaufler [replied](<https://lwn.net/ml/linux-kernel/07e92045-2d80-8573-4d36-643deeaff9ec@schaufler-ca.com/>) with an example (`/dev/null`), where this is not the case, saying that many such examples exist. Lutomirski then [agreed](<https://lwn.net/ml/linux-kernel/CALCETrVuNRPgEzv-XY4M9m6sEsCiRHxPenN_MpcMYc1h26vVwQ@mail.gmail.com/>) that notifications between unrelated processes should not be allowed for a file like `/dev/null`. That opens the door for a renewed discussion on the security policies around notifications. 

This understanding has not, yet, led to a full agreement about what those policies should be. It would not be surprising if a full consensus were to take a while to emerge; this is a complex new API with new security implications for every subsystem that submits events to it. One generally wants to have the security story figured out before something like this is released in a mainline kernel. So this work may or may not find its way into 5.3, but it does appear to have a reasonable chance of avoiding the fate of many other generalized event-notification mechanisms and going upstream eventually.
