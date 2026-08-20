---
title: "KS2010: ABI status for tracepoints"
url: https://lwn.net/Articles/412685/
date: "November 2, 2010"
category: "Development model-User-space ABI; Tracing-ABI issues"
author: "By Jonathan Corbet November 2, 2010 2010 Kernel Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 2, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

Whether tracepoints should be a part of the user-space binary interface is a topic which has been [covered here](<https://lwn.net/Articles/401769/>) a number of times in the past. When the 2010 Kernel Summit took up the topic, many assumed that the result would be a controversial session. Instead, the developers present came rather quickly to a consensus on how the issue should be resolved. 

Arjan van de Ven started with a demonstration of a new version of PowerTop which provides all kinds of information on just how systems are using power. It depends heavily on tracepoints for the source information. He wants to make useful tools like this, but he cannot do it if those tracepoints are not seen as part of the ABI. That interface needs to not break, or the tools will not be useful. 

The other leader, Steve Rostedt, started by saying that tracepoints had been put into debugfs for a reason. Tracepoints reach deep into the internal state of the kernel; setting them in stone could limit its future development. But he recognized the need for at least some tracepoints to be relatively stable, so he proposed that there should be two types of tracepoints. One, intended mainly for in-field debugging, would have no ABI guarantee. The other set of tracepoints, aimed at "high-level questions," would be stabilized in some way. 

Exactly what "stable" means is not entirely clear; the specific format of a tracepoint could change, perhaps, but the relevant fields would remain and the associated format description would tell applications how to find the information the need. There seemed to be a consensus that requiring applications to use the format description is reasonable. Ted Ts'o pointed out, though, that a lot of tools have trouble with complicated formats. So maybe stable tracepoints should only use simpler formats. 

Ted also wondered about just how stable tracepoints would be marked as such. We could impose documentation requirements, but those have not always been observed or enforced in the past. They could be documented in the header file somewhere. Or, possibly, stable tracepoints could be moved out of debugfs entirely, probably to a sysfs subdirectory. That last idea proved popular, and is how things are likely to be done. 

Linus was receptive to all of these ideas, but he did have one requirement: there are to be no stable tracepoints in drivers or filesystems. His long experience with `ioctl()` calls tells him that driver developers will never get this right and will not be able to maintain stable tracepoints in the long term. He would also rather not see them in architecture-specific code either. One idea which went over fairly well was to make stable tracepoints depend on a symbol which is not exported to modules. That would prevent stable tracepoints from appearing in any code which can be built as a module. There may also be a requirement that all stable tracepoints be designated in a single, central source file to make it evident when new ones are added. 

All of this seems likely to pass; there were no voices raised in disagreement. There will still be some interesting questions, though. Steve showed a tracepoint from the scheduler which, he said, would be a good candidate for stable status - it fires when the scheduler switches from one process to another. Peter Zijlstra objected to some of the contents of that tracepoint; it output information which, he says, does not belong in a stable tracepoint. For example, there is priority information provided, but, if the deadline scheduler is merged, there will be processes with no priorities. So, while stable tracepoints may be in the future, they are still likely to provide plenty to argue about. 

[Next: The core kernel vision](<https://lwn.net/Articles/412687/>).
