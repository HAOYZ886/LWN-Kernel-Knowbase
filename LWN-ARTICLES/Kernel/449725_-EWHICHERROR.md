---
title: "-EWHICHERROR?"
url: https://lwn.net/Articles/449725/
date: "June 29, 2011"
category: "Development model-User-space ABI; Error codes"
author: "By Jonathan Corbet June 29, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 29, 2011

Users of the Video4Linux2 API know that it is a rather complicated one, involving some 91 different `ioctl()` commands. The error-reporting side of the API is much simpler, though; if something goes wrong, the application is almost certain to get `EINVAL` back. That error can be trying to tell user space that the device is in the wrong state, that some parameter was out of range, or, simply, that the requested command has not been implemented. Needless to say, it can be hard for developers to figure out what is really going on. 

V4L2 maintainer Mauro Carvalho Chehab recently posted [a patch](<https://lwn.net/Articles/449726/>) to change the return code to `ENOIOCTLCMD` in cases where the underlying driver has not actually implemented the requested command. That change would at least distinguish one set of problems - except that the VFS code silently translates `ENOIOCTLCMD` to `EINVAL` before returning to user space. So, from the point of view of the application, nothing changes. 

Interestingly, the rules for what is supposed to happen in this situation are relatively clear: if an `ioctl()` command has not been implemented, the kernel should return `ENOTTY`. Some parts of the kernel follow that convention, while others don't. This is not a new or Linux-specific problem; as Linus [put it](<https://lwn.net/Articles/449727/>): ""The EINVAL thing goes way back, and is a disaster. It predates Linux itself, as far as I can tell."" He has suggested simply changing `ENOIOCTLCMD` to `ENOTTY` across the kernel and seeing what happens. 

What happens, of course, is that the user-space ABI changes. It is entirely possible that, somewhere out there, some program depends on getting `EINVAL` for a missing `ioctl()` function and will break if the return code changes. There is only one way to find out for sure: make the change and see what happens. Mauro [reports](<https://lwn.net/Articles/449729/>) that making that change within V4L2 does not seem to break things, so chances are good that change will find its way into 3.1. A tree-wide change could have much wider implications; whether somebody will find the courage to try that remains to be seen.
