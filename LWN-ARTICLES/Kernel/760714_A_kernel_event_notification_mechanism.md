---
title: A kernel event notification mechanism
url: https://lwn.net/Articles/760714/
date: "July 25, 2018"
category: Events reporting
author: "By Jonathan Corbet July 25, 2018"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 25, 2018

The kernel has a range of mechanisms for notifying user space when something of interest happens. These include [dnotify and inotify](<https://lwn.net/Articles/604686/>) for filesystem events, signals, `poll()`, tracepoints, uevents, and more. One might think that there would be little need for yet another, but there are still events of interest that user space can only learn about by polling. In an attempt to fix this problem, David Howells, not content with his recent attempt to [add seven new system calls](<https://lwn.net/Articles/759499/>) for filesystem mounting, has put forward [a proposal](<https://lwn.net/Articles/760596/>) for a general-purpose event notification mechanism for Linux. 

The immediate use case for this mechanism is to provide user space with notifications whenever the filesystem layout changes as the result of mount and unmount operations. That information can be had now by repeatedly reading `/proc/mounts`, but doing so evidently can impair the performance of the system. The patch set also provides for superblock-level events, such as I/O errors, filesystems running out of space, or processes running into disk quotas. Finally, the ability to watch for changes to in-kernel keys or keyrings is also included. 

The BSD world has long had the [`kqueue()` and `kevent()`](<https://www.freebsd.org/cgi/man.cgi?query=kevent&apropos=0&sektion=0&manpath=FreeBSD+9.0-RELEASE&arch=default&format=html>) system calls for this purpose. Naturally, the mechanism proposed by Howells looks nothing like that API. It is, instead, seemingly designed for performance even with high event rates; to get there, user space must set up and manage a circular buffer that is used to transfer events from the kernel. (As an aside, the kernel already has a whole set of circular-buffer mechanisms for perf events, ftrace events, network packets, and more. This patch set adds yet another. It would have been nice, years ago, to create a single abstraction for these buffers so that a set of library functions could be provided to work with all of them, but that ship sailed some time ago.) 

#### Setting up the event buffer

There is no system call dedicated to setting up the event buffer; instead, the first step is to open a special device (`/dev/watch_queue`) for that purpose. User space then uses `ioctl()` to configure this buffer, starting with the `IOC_WATCH_QUEUE_SET_SIZE` command to set its size (in pages). The application will need to call `mmap()` on the device file descriptor to map the event buffer into its address space. 

Then, the application needs to arrange for events of interest to be delivered into this buffer. There are actually two separate tasks that must be done here: asking for events to be delivered, and configuring a filter to control which events actually make it into the ring buffer. Requesting delivery is dependent on the event type. For events related to keys, there is a new command for the [`keyctl()`](<http://man7.org/linux/man-pages/man2/keyctl.2.html>) system call: 

```
int keyctl(KEYCTL_WATCH_KEY, key_serial_t id, int buffer,
                   unsigned char watch_id);
```

Where `id` identifies the key of interest, `buffer` is the file descriptor for the event buffer, and `watch_id` is an eight-bit identifier that will appear in any generated events. For filesystem topology events, a new system call is used: 

```
int mount_notify(int dfd, const char *path, unsigned int flags,
        		     int buffer, unsigned char watch_id);
```

Here, `dfd` and `path` identify the mount point, `flags` is one of the `AT_*` flags controlling how `path` is followed, `buffer` is the file descriptor for the event buffer, and `watch_id` is the user-supplied identifier. For superblock events, a similar system call has been added: 

```
int sb_notify(int dfd, const char *path, unsigned int flags,
        		  int buffer, unsigned char watch_id);
```

No doubt there will be other types of notifications added in the future if this mechanism makes it into the mainline kernel. 

Each of the calls above will generate notifications for a number of different event types. For example, superblock events in the current patch set include "filesystem was toggled between read/write and read-only", "I/O error", "disk quota exceeded", and "network status change". The requesting application may not be interested in all of these event types. Getting the right ones requires setting up a filter, which is done by filling in a `watch_notification_filter` structure: 

```
struct watch_notification_type_filter {
    	__u32	type;			/* Type to apply filter to */
    	__u32	info_filter;		/* Filter on watch_notification::info */
    	__u32	info_mask;		/* Mask of relevant bits in info_filter */
    	__u32	subtype_filter[8];	/* Bitmask of subtypes to filter on */
        };
    
        struct watch_notification_filter {
    	__u32	nr_filters;		/* Number of filters */
    	__u32	__reserved;		/* Must be 0 */
    	struct watch_notification_type_filter filters[];
        };
```

For each entry in the `filters` array, `type` identifies the subsystem type of the event (`WATCH_TYPE_MOUNT_NOTIFY`, `WATCH_TYPE_KEY_NOTIFY`, or `WATCH_TYPE_SB_NOTIFY` in the current patch set), `subtype_filter` is a bitmask indicating the specific events that the application is interested in — `notify_key_instantiated`, `notify_mount_unmount`, or `notify_superblock_error`, for example. The `info_filter` field can be used to further filter on event-specific information; it can be used to catch mount-point transitions to read/write, for example, while ignoring transitions to read-only. 

The `IOC_WATCH_QUEUE_SET_FILTER` `ioctl()` command must be used to set the filter once the description is ready. At that point, events can be delivered into the circular buffer. 

#### Receiving events

The buffer itself is defined with this structure: 

```
struct watch_queue_buffer {
    	union {
    	    /* The first few entries are special, containing the
    	     * ring management variables.
    	     */
    	    struct {
    		struct watch_notification watch; /* WATCH_TYPE_SKIP */
    		volatile __u32	head;		/* Ring head index */
    		volatile __u32	tail;		/* Ring tail index */
    		__u32		mask;		/* Ring index mask */
    	    } meta;
    	    struct watch_notification slots[0];
    	};
        };
```

The `union` setup may look a bit strange; it is designed so that the `meta` information looks like a special type of event entry that will be automatically skipped over by code reading through the buffer. The `head` index points to the first free slot (where the kernel will write the next event), while `tail` points to the first available event. User space can adjust the `tail` pointer only. If `head` and `tail` are equal, the buffer is empty. 

The actual events look like: 

```
struct watch_notification {
    	__u32			type:24;
    	__u32			subtype:8;
    	__u32			info;
        };
```

The `type` and `subtype` fields describe the specific event; `info` is rather more complicated, though, being made up of several fields that must be masked to be used. For example, events can take up more than one slot in the buffer; masking with `WATCH_INFO_LENGTH` yields the number of slots used. Use `WATCH_INFO_ID` to get the `watch_id` value provided when the event was requested. Also crammed into `info` are flags to indicate buffer overruns or lost events, and a bunch of event-specific flags. The `info_filter` in the filter set up by user space can filter on most of the fields within `info`. 

Once all that is set up, it's just a matter of watching `head` and `tail` (using appropriate barrier operations) to detect when there are events in the structure to be consumed. It is also possible to call `poll()` on the buffer file descriptor to wait for new events to arrive. 

This is the first posting of this patch set, and the work is clearly still changing quickly; this can be observed by noting that the API descriptions in the changelogs are seemingly from a previous version and do not match what the code actually implements. Anybody interested in how this API looks from user space can look at [this example program](<https://lwn.net/ml/linux-kernel/153235957887.32640.8025222191782276269.stgit@warthog.procyon.org.uk/>) included with the patch set. About the only comment so far has been [from Casey Schaufler](<https://lwn.net/ml/linux-kernel/675e5c24-36ef-4cc5-846c-1414c1195d85@schaufler-ca.com/>), who is concerned about how the mechanism interacts with security modules and keeping users from receiving events that they shouldn't. 

These patches are clearly intended to create a general-purpose mechanism that could be used throughout the kernel, so they will need a fair amount of review before they can be accepted. Changes seem likely. If the inevitable concerns can be addressed, Linux may yet have a general event-notification mechanism, even if we'll never get `kevent()` and `kqueue()`.
