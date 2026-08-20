---
title: On the merging of ktimers
url: https://lwn.net/Articles/156325/
date: "October 19, 2005"
category: Development model; Timers
author: 
---

LWN [looked at the ktimers patch](<http://lwn.net/Articles/152436/>) about one month ago. Work continues on the new kernel timer mechanism; the [latest version](<http://lwn.net/Articles/155862/>) of the patch includes a new "clockevents" abstraction intended to make high-resolution timer support easier to implement in an architecture-independent way. The patch appears to be coming together well, and there has been little in the way of criticism. 

...with the exception of one observer, who has kept up a steady stream of complaints about the new mechanism. His objections include the name (he would rather see "process timers" than "ktimers"), the use of high-resolution time within the kernel, and various "unnecessary complexities." The discussion has been mostly unfruitful, to the point that the normally even-keeled Ingo Molnar tried to end it with a [shut up and show me the code challenge](<https://lwn.net/Articles/156327/>). That led Andrew Morton to state that "show me the code" is no longer an acceptable arguing point for kernel discussions, and that the objections should be addressed regardless. 

Getting a handle on the objections has proved hard; it is not clear that the person in question (Roman Zippel) truly understands the patches. One bit of the discussion is worth a look, however. It has been repeatedly pointed out that the existing kernel timer mechanism is optimized for timeouts which rarely actually expire, while ktimers are _expected_ to expire. Roman [claimed](<https://lwn.net/Articles/156328/>): 

Whether the timer event is delivered or not is completely unimportant, as at some point the event has to be removed anyway, so that optimizing a timer for (non)delivery is complete nonsense. 

This claim led to [a required-reading response from Ingo](<https://lwn.net/Articles/156329/>) on the history of the kernel timer mechanism and why optimizing for delivery (or the lack thereof) is _not_ nonsense. That particular branch of the discussion, at least, should not need to go much further. 

Andrew Morton has, in the past, stated that he would be highly reluctant to merge new code over the objections of a developer. The need to address all objections can be highly frustrating to kernel hackers, especially when new complaints seem to keep turning up as the old ones are resolved. The result of this process, when it works well, can be a stronger kernel. But it can also be the delaying of useful code which few people have problems with. It is starting to look like that may be the outcome in the ktimers case; the code will almost certainly be merged in the end, perhaps with almost no changes resulting from the current discussion.
