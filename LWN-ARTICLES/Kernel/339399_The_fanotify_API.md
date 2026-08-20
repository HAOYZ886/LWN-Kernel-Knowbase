---
title: The fanotify API
url: https://lwn.net/Articles/339399/
date: "July 1, 2009"
category: "Security-Security technologies; fanotify"
author: "By Jonathan Corbet July 1, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 1, 2009

One of the features merged for 2.6.31 is the "fsnotify" file event notification framework. Fsnotify serves as a new, common underpinning for the inotify and dnotify APIs, simplifying the code considerably. But this simplification, as welcome as it is, was never the real purpose behind fsnotify. Instead, fsnotify exists to serve as the support structure for [fanotify](<http://lwn.net/Articles/339253/>), the "fscking all notification system," which has now been posted for further review. 

Fanotify was once known as [TALPA](<http://lwn.net/Articles/292872/>); its main purpose is to enable the implementation of malware scanners on Linux systems. When TALPA was first proposed, it ran into criticism on a number of fronts, not the least of which being a general disdain for malware scanning as a security technique. The sad fact of the matter, though, is that a number of customers require this functionality, so a market for such products on Linux exists. Thus far, scanning products for Linux have relied on a number of distasteful techniques, including hooking into the system call table or the loading of binary-only security modules. Fanotify, it is hoped, will help to wean these products off of such hacks and get them out of the kernel altogether. 

The user-space API used by fanotify is, to your editor's eye, a little strange. An fanotify application starts by opening a socket with the new `PF_FANOTIFY` protocol family. This socket must then be bound to an "address" described this way: 

```
struct fanotify_addr {
    	sa_family_t family;
    	__u32 priority;
    	__u32 group_num;
    	__u32 mask;
    	__u32 timeout;
    	__u32 unused[16];
        };
```

The `family` field should be `AF_FANOTIFY`. The `priority` field is used to determine which socket gets a specific event if more than one fanotify socket exists; lower priority values win. The `group_num` is used by the fsnotify layer to identify a group of event listeners. The `timeout` field currently appears to be unused. Finally, `mask` describes the events that the application is interested in hearing about: 

  * `FAN_ACCESS`: every file access. 
  * `FAN_MODIFY`: file modifications. 
  * `FAN_CLOSE`: when files are closed. 
  * `FAN_OPEN`: `open()` calls. 
  * `FAN_ACCESS_PERM`: like `FAN_ACCESS`, except that the process trying to access the file is put on hold while the fanotify client decides whether to allow the operation. 
  * `FAN_OPEN_PERM`: like `FAN_OPEN`, but with the permission check. 
  * `FAN_EVENT_ON_CHILD`: the caller is interested in events on full directory hierarchies. 
  * `FAN_GLOBAL_LISTENER`: notify for events on all files in the system. 

Once the socket has been bound, the application can learn about filesystem activity using the well-known event-reading system call `getsockopt()`. A call to `getsockopt()` with `SOL_FANOTIFY` as the level and `FANOTIFY_GET_EVENT` as the option will return one or more structures like this: 

```
struct fanotify_event_metadata {
    	__u32 event_len;
    	__s32 fd;
    	__u32 mask;
    	__u32 f_flags;
    	pid_t pid;
    	pid_t tgid;
    	__u64 cookie;
        };
```

Here, `fd` is an open, read-only file descriptor for the file in question, `mask` describes the event (using the flags described above), `f_flags` is a copy of the flags provided by the process trying to access the file, and `pid` and `tgid` identify that process (in a namespace-unaware way, currently). If the event is one requiring permission from the application, `cookie` will contain a value which can be used to grant or deny that permission. 

Note that the provided file descriptor should eventually be closed; otherwise these file descriptors are likely to accumulate rather quickly. 

When access decisions are being made, the application must notify the kernel with a call to `setsockopt()` using the `FANOTIFY_ACCESS_RESPONSE` option and a structure like: 

```
struct fanotify_so_access {
    	__u64 cookie;
    	__u32 response;
        };
```

The `cookie` value from the event should be provided, and `response` should be one of `FAN_ALLOW` or `FAN_DENY`. If the application does not respond within five seconds, the kernel will allow the action to proceed. Five seconds should be sufficient for file scanning, but it could be a problem with some other possible applications of fanotify, such as hierarchical storage management systems. Fanotify developer Eric Paris [notes](<https://lwn.net/Articles/339413/>) that a future option allowing the response to be delayed indefinitely will probably be added at some point. 

It is possible to adjust the set of files subject to notifications with the `FANOTIFY_SET_MARK`, `FANOTIFY_REMOVE_MARK`, and `FANOTIFY_CLEAR_MARKS` operations. If the `FAN_GLOBAL_LISTENER` option was provided at bind time, then all files are "marked" at the outset; `FANOTIFY_REMOVE_MARK` can be used to prune those which are not interesting. Otherwise at least one `FANOTIFY_SET_MARK` call must be made before events will be returned. 

Some details have been left out, but the above discussion covers the core parts of the fanotify API. Comments on this posting have been relatively scarce; opposition to this feature seems to have faded away over the last year or so. What's left is getting the API right; your editor suspects that the use of `getsockopt()` as an event retrieval interface may raise a few eyebrows sooner or later. But, once that's ironed out, chances are good that Linux will be well on the way toward having a general file access notification and permission interface.
