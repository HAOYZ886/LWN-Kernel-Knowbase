---
title: Handling I/O errors in the kernel
url: https://lwn.net/Articles/757123/
date: "June 12, 2018"
category: "Block layer-Error handling"
author: "By Jake Edge June 12, 2018 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
June 12, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

The kernel's handling of I/O errors was the topic of a discussion led by Matthew Wilcox at the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM) in a combined storage and filesystem track session. At the start, he asked: "how is our error handling and what do we plan to do about it?" That led to a discussion between the developers present on the kinds of errors that can occur and on ways to handle them. 

Jeff Layton said that one basic problem occurs when there is an error during writeback; an application can read the block where the error occurred and get the old data without any kind of error. If the error was transient, data is lost. And if it is a permanent error, different filesystems handle it differently, which he thinks is a problem. Dave Chinner said that in order to have consistent behavior across filesystems, there needs to be a definition of what that behavior should be. There is a need to distinguish between transient and permanent failures and to create a taxonomy of how to deal with each type. 

Kent Overstreet said that transient errors should be handled by the lower layers so that filesystems never see a transient write error. But others brought up transient errors that might make sense to return to the filesystems (e.g. no blocks on a thinly provisioned device, some kind of authorization problem or expiration). So, Chinner said, that might add another class of error beyond transient and permanent: user-response required. 

Because of thin provisioning, `ENOSPC` can be a transient or permanent error, but what about transient errors that last so long they should be treated as permanent, Layton asked. Overstreet said there should be a global setting for how long operations with transient errors should be retried. XFS has multiple settings, Chinner said: try forever, do not retry, or a timeout coupled with a number of retries. 

It might be nice if there were a single knob for administrators to set the behavior they want, Ted Ts'o said. But it would need to be a per-device setting and administrators would probably want to control it per filesystem as well, so it really wouldn't be a single knob. Chinner said that is as it should be, there are complicated systems out there and administrators need to be able to tweak things at every level. Another attendee said that different transports have different characteristics, so treating USB, PCI, and Fibre Channel devices the same does not really make sense. 

But you do want a random user to be able to plug a USB drive into their laptop and get reasonable settings, Ts'o said. That's where defaults come in, Chinner said. Layton said there was a need for lots of knobs, but sane defaults. Overstreet said it would help if there was some consistency in where the knobs are, though. 

Wilcox asked what changes could come before next year's LSFMM. Chinner said that XFS will probably extend its metadata error handling to its data blocks over the next year. Ts'o wondered if other filesystems should follow XFS's lead on that. Layton asked what XFS would do for pages with transient errors during writeback and whether they would be left in the dirty state, unlike what is done today. Chinner said that it would leave the pages dirty for transient errors that are still being retried. 

Layton said that informing user space will be tricky; if the writes are still being retried, that means they could still eventually fail. A call to `fsync()` is an opportunity to throw the dirty pages away (by marking them as clean) since the error can be reported to user space at that point. Whatever is done, it should be done in the VFS layer so that each filesystem does not have to do it, Wilcox said. 

In a final exchange as the time for the session was expiring, Layton wondered about permanent errors. If the application does a read after a failed write that effectively throws away the changes by marking the page clean, it might get the new data, which won't make it to permanent storage. Chinner suggested that perhaps the read operation could return an error under those conditions.
