---
title: Monitoring mount operations
url: https://lwn.net/Articles/932648/
date: "May 24, 2023"
category: "Filesystems-Mounting; fanotify"
author: "By Jake Edge May 24, 2023 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
May 24, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Amir Goldstein kicked off a session on monitoring mounts at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>). In particular, there are problems when trying to efficiently monitor "a very large number of mounts in a mount namespace"; some user-space programs need an accurate view of the mount tree without having to constantly parse `/proc/mounts` or the like. There are a number of questions to be answered, including what the API should look like and what entity should be watched in order to get notifications of new mount operations. 

It is trivial, he said, to add a notification for unmount events, but the corresponding event for a new mount is trickier, since it is not clear where, exactly, the watch for that event should be placed. It could be placed on the user or mount namespace of interest; another idea would be to choose a directory and monitor all of the mounts that happen on it and any of its subdirectories recursively. David Howells said that he has implemented something for getting mount notifications; the watch is placed on the mount namespace. Miklos Szeredi said that each namespace has its own mount tree and each mount has a 32-bit ID that gets assigned to it, but those cannot reliably be used to uniquely identify a particular mount because they can be reused during a given boot of the system. Howells said that he added a 64-bit counter that could be used for that purpose, though it will "eventually get reused" as well. 

Howells was asked about patches, which he said he had [posted](<https://lwn.net/ml/linux-kernel/159645997768.1779777.8286723139418624756.stgit@warthog.procyon.org.uk/>) a while back. Szeredi pointed out that those patches were not for [fanotify](<https://www.kernel.org/doc/html/latest/admin-guide/filesystem-monitoring.html>) support, but were instead for the [watch queue](<https://docs.kernel.org/core-api/watch_queue.html>); it is the same general concept, however, he said. Christian Brauner thought that the notification piece should be separated from the [`fsinfo()` effort](<https://lwn.net/Articles/829212/>). 

The problem, Howells said, is that the notification queue can overflow, which means that events, such as mount and unmount operations, would get lost. Howells mentioned that currently tools have to parse (and poll) `/proc/mounts` in order to find out the status of mounts and unmounts, which is not particularly efficient. Brauner noted that he had invited Lennart Poettering to the talk, since systemd would be one of the eventual users of any new feature of this sort, so he asked Poettering about systemd's needs in this area. 

Right now, systemd parses `/proc/self/mountinfo`, "which, of course, is terrible", Poettering said. He is not particularly concerned if events get dropped, as long as there is a way to figure out what has happened; some kind of unambiguous indication that events have been dropped coupled with an API for systemd to get the current status when it needs to do so would be ideal. He would like a facility that provides the immediate child mounts for a given mount along with mount-related events for those children. Howells said that Ian Kent had created a [patch set implementing mount watching for systemd](<https://lists.freedesktop.org/archives/systemd-devel/2020-July/044914.html>) using `fsinfo()` and the watch queue notifications. 

Brauner asked if the feature needed to be added to fanotify for systemd's use, but Poettering said that he did not care. His main concern is in getting notified when events are lost, so that systemd can take some action to update its state; it would be great if the lost-event notification narrowed down where in the mount tree the lost event(s) came from. For systemd's use case, it would be better to get events for a particular subtree, rather than the whole system, because it normally is only concerned with a subset of the full mount tree. 

Jeff Layton asked about the systemd use case for this information. Poettering responded that there are many systemd services that need to wait for mount activity of some form (e.g. at boot time, MySQL needs to wait for the filesystem where its files reside). Much of systemd's dependency processing for services depends on an accurate picture of the state of the system, including mounts. 

Goldstein said that he was unsure how to report the occurrence of a [tucked mount](<https://lwn.net/Articles/927491/>), which is a mechanism aimed at cleanly replacing an overlay mount. Brauner said that he was no longer "allowed to call it that"; there is another interpretation of that term, which he was unaware of until "friendly people on social media" pointed it out to him. They suggested using "beneath" to describe the type of mount. There is also, of course, the danger of mistyping the previous term, he said. 

There was some discussion of a way to retrieve the immediate child mounts, as Poettering requested, but that will require a unique mount ID, Brauner said. After some roundabout discussion about mount-related APIs and the concerns that would need to be kept in mind, worries about a mount-ID overflow were raised. Layton pointed out that a 64-bit counter that gets incremented every nanosecond will take more than 500 years to overflow, so "we're never going to overflow at 64 bits". 

There may be problems with exposing those 64-bit values to user-space programs that expect only a 32-bit mount-ID, however. In fact, Poettering checked the systemd code and it "knows" that the mount-IDs are 32-bits in size. Howells said that the existing mount-ID is "recycled, too small, people assume it is too small", so something new that is defined to be 64-bits is needed. Poettering suggested using [UUID](<https://en.wikipedia.org/wiki/Universally_unique_identifier>)s "and the problem goes away", he said, to chuckles around the room. As time expired, things kind of trailed off; it is clear that there is more work needed before anything is likely to go upstream.
