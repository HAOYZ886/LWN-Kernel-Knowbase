---
title: Another new ABI for fanotify
url: https://lwn.net/Articles/360955/
date: "November 11, 2009"
category: fanotify
author: "By Jonathan Corbet November 11, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 11, 2009

"Fanotify" is a much-revised system for providing filesystem event notifications to user space, and, possibly, allowing user space to block `open()` operations on specific files. The intended use case is malware-scanning utilities, but there are others: hierarchical storage has been cited as one possibility. This code has had a long, hard path into the kernel for a couple of reasons: kernel developers are not big fans of malware scanning, and nailing down the user-space ABI has been challenging. 

The first obstacle has been more-or-less overcome. Even developers who think that malware scanning is the worst sort of security snake oil can agree that having these utilities use a well-defined kernel interface is better than having them employ nasty tricks like hooking into the system call table. ABI difficulties can be harder to overcome, though. With [the latest fanotify posting](<http://lwn.net/Articles/359840/>), developer Eric Paris may have resolved this issue for at least a portion of the fanotify functionality. 

The new version does away with the novel interface using `setsockopt()` in favor of a couple of new system calls. The first of these is `fanotify_init()`: 

```
int fanotify_init(unsigned int flags, unsigned int event_f_flags,
    		      int priority);
```

This system call initializes the fanotify subsystem, returning a file descriptor which is used for further operations. There are two `flags` values implemented: `FAN_NONBLOCK` creates a nonblocking file descriptor, and `FAN_CLOEXEC` sets the close-on-exec flag. Currently, `event_f_flags` and `priority` are unused; they should be set to zero. 

Management of notification events is then done with `fanotify_mark()`: 

```
int fanotify_mark(int fanotify_fd, unsigned int flags,
    		      int dfd, const char *pathname, u64 mask,
    		      u64 ignored_mask);
```

This call is used to "mark" specific parts of the filesystem hierarchy, indicating an interest in events involving those files. `fanotify_fd` is the file descriptor returned by `fanotify_init()`. The `flags` parameter must be one of `FAN_MARK_ADD` or `FAN_MARK_REMOVE`, indicating whether this call adds new marks or removes existing ones; there are also a couple of flags to control following of symbolic links and the marking of directories (without their contents). 

The file(s) to be marked are determined by `dfd` and `pathname`; these parameters work much like in any of the `*at()` system calls. If `dfd` is `AT_FDCWD`, the `pathname` is resolved using the current working directory. If, instead, `dfd` points to a directory, the `pathname` lookup starts at that directory. If `pathname` is null, though, then `dfd` is interpreted as the actual object to mark. 

Finally, `mask` and `ignored_mask` control which events are reported. To generate a specific event, a file must have the appropriate flag set in `mask` and clear in `ignored_mask`. The flags are `FAN_ACCESS` (file access), `FAN_MODIFY` (a file is modified), `FAN_CLOSE_WRITE` (a writable file has been closed), `FAN_CLOSE_NOWRITE` (a read-only file has been closed), `FAN_OPEN` (a file has been opened), and `FAN_EVENT_ON_CHILD` (events on children of a directory). There is also a `FAN_Q_OVERFLOW` event for event queue overflows, but that is not currently implemented. 

Once files have been marked, the application can simply read from the fanotify file descriptor to get events. The events look like: 

```
struct fanotify_event_metadata {
    	__u32 event_len;
    	__u32 vers;
    	__s32 fd;
    	__u64 mask;
        };
```

Here, `event_len` is the length of the structure, `vers` indicates which version of fanotify generated the structure, `fd` is an open file descriptor for the object being accessed, and `mask` describes what is actually happening. 

There is one crucial component missing in these patches: there is no way for the fanotify user to react to these events. In particular, the ability to block an `open()` call, a core part of the malware-scanning process, is missing. That, presumably, is to be added in a future revision. Meanwhile, Eric has requested permission to put the notification code into linux-next, presumably with a 2.6.33 merge in mind. As of this writing, objections have not been forthcoming.
