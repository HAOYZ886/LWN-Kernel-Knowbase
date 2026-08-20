---
title: Watching filesystem events with inotify
url: https://lwn.net/Articles/104343/
date: "September 29, 2004"
category: Events reporting; Inotify
author: 
---

It is not uncommon for applications to want to know when something happens within a subtree of a filesystem. File managers are the most obvious example; if an application creates a new file within a directory represented in a file manager, users really like to see that new file show up, quickly. One could also imagine other sorts of applications - such as security monitoring code or just daemons wanting to know when their configuration files have changed - which can benefit from being told about filesystem activity. 

The Linux mechanism for communicating filesystem events to user space is called "dnotify." A program watches a directory by opening it, then issuing a `fcntl(F_NOTIFY)` call. Thereafter, changes in that directory will result in a `SIGIO` signal being sent to the process, which can then dig through its cached information and try to figure out just what happened. People like to complain about dnotify; the interface is ugly (signals are a pain), it is hard to figure out what the changes are, it requires keeping files open and thus blocks the unmounting of removable media, etc. So there has long been interest in a replacement. 

The most visible effort in that direction is inotify, which has been under development (by John McCutchan) for some time now; recently Robert Love has jumped in to help the project along. [inotify 0.11](<https://lwn.net/Articles/104312/>) was released on September 28, and an increasingly strong push is being made to get it included into -mm for wider exposure and testing. 

inotify works through a new character pseudo-device. Any application which wants to monitor filesystem activity need only open `/dev/inotify` and issue one of two `ioctl()` commands to it: 

`INOTIFY_WATCH`
    This call provides a filename and a mask of desired events; inotify will begin watching the given file (or directory) for activity. 

`INOTIFY_IGNORE`
    This call will stop the stream of events for the given file.

Quite a few possible events can be watched for: `IN_ACCESS` (the file was accessed), `IN_MODIFY` (the file was changed), `IN_ATTRIB` (file attributes changed), `IN_OPEN` and `IN_CLOSE` (for open and close events), `IN_MOVED_FROM` and `IN_MOVED_TO` (when files are renamed), `IN_CREATE_SUBDIR` and `IN_DELETE_SUBDIR` (creation and deletion of subdirectories), `IN_CREATE_FILE` and `IN_DELETE_FILE` (creation and deletion of files within a directory), `IN_DELETE_SELF` (when a monitored file is deleted), `IN_UNMOUNT` (when the filesystem containing the file is unmounted), and a couple of others. The events themselves are obtained by simply reading from the device. Thus a program can block on the device itself, or use `poll()` to incorporate notifications into a larger event-processing loop. No signals are involved. 

The actual implementation of inotify is relatively simple. The in-core inode structure is augmented with a linked list of processes interested in events involving that inode. When an `INOTIFY_WATCH` call is made, an entry is made in the corresponding list (and the inode is pinned into memory for the duration). Various parts of the filesystem code get an extra `inotify_inode_queue_event()` call when an action succeeds. The rest is just the usual overhead of maintaining lists of events for processes, waking those processes up when new events arrive, etc. 

While most interest and activity seems to be around inotify, it is not the only dnotify replacement in circulation; [nonotify](<https://lwn.net/Articles/98922/>) is an alternative. There are also some remaining issues about the interface exported by inotify. It has been suggested that the inotify `ioctl()` calls should take file descriptors rather than file names; that change would eliminate problems in dealing with long file names and would also make access control checks happen automatically. The interface would have to be done in such a way that the application could close the file and still receive events, though; otherwise dnotify's problems with unmount blocking and excessive use of file descriptors would just come back again. These issues notwithstanding, inotify looks like it is headed for inclusion into a mainline kernel in the not-too-distant future.
