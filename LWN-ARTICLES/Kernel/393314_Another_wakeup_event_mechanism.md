---
title: Another wakeup event mechanism
url: https://lwn.net/Articles/393314/
date: "June 23, 2010"
category: "Android; Power management-Opportunistic suspend"
author: "By Jonathan Corbet June 23, 2010"
---

By **Jonathan Corbet**  
June 23, 2010

The suspend blocker discussion has faded away for the time being - a situation which has drawn few complaints. Developers are still thinking about the underlying problems addressed by suspend blockers, though, as can be seen from [this patch](<http://lwn.net/Articles/392897/>) by Rafael Wysocki. Rafael is trying to solve the problem of "wakeup events" (events requiring action which would wake a suspended device) being lost if they show up while the system is suspending. 

In Rafael's approach, there would be a new sysfs attribute called `/sys/power/wakeup_count`; it would contain the number of wakeup events seen by the system so far. Any process can read this attribute at any time to obtain this count; privileged processes can also write a count back to the file. There is a twist, though: if the count written to the file does not match the count which would be read from it, the write will fail. A write also triggers a mechanism whereby any subsequent wakeup events will cause an attempted suspend operation to abort. 

As with some other scenarios which have been posted, Rafael is assuming the existence of a user-space power management daemon which would decide when to suspend the system. This decision would be made when the daemon knows that no important user-space program has work to do. Without extra help, though, there will be a window between the time that the daemon decides to suspend the system and when that suspend actually happens; a wakeup event which arrives within that window could be lost, or at least improperly delayed until after the system is resumed again. But, with the wakeup_count mechanism, the kernel would notice when this kind of race had happened and abort the suspend process, allowing user space to process the new event. 

For this mechanism to work, the kernel must be able to count wakeup events; that, in turn, requires sprinkling calls to a new `pm_wakeup_event()` function into drivers which can generate such events. So internally it doesn't look all that different from suspend blockers. Some of the comments have suggested that the scheme is quite similar to suspend blockers on the user-space side too, though Rafael believes that it avoids the aspects of that API which generated the most criticism. Regardless, reviews were mixed, and Android developer Arve Hjønnevåg [thinks](<https://lwn.net/Articles/393321/>) that this approach will not meet that project's needs. So this discussion probably has more rounds to go in the future.
