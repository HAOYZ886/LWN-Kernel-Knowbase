---
title: A tempest in a tty pot
url: https://lwn.net/Articles/343828/
date: "July 29, 2009"
category: "Development model-Kernel quality; TTY layer"
author: "By Jonathan Corbet July 29, 2009"
---

By **Jonathan Corbet**  
July 29, 2009

There are dark areas of the kernel where only the bravest hackers dare to tread. Places where the code is twisted, the requirements are complex, and everything depends on ancient code which has seen little change over the years because even the most qualified developers fear the consequences. Arguably, no part of the kernel is darker and scarier than the serial terminal (TTY) code. Recently, this code was getting a much-needed update, but it now appears that a disconnect within the community has brought that work to a halt and thrown TTY back into the "unmaintained" column - at a time when that code has known regressions in the 2.6.31-rc kernel. 

At a first glance, the TTY layer wouldn't seem like it should be all that challenging. It is, after all, just a simple char device which is charged with transferring byte-oriented data streams between two well-defined points. But the problem is harder than it looks. Much of the TTY code has roots in ancient hardware implementing the RS-232 standard - one of the loosest, most variable standards out there. TTY drivers also have to monitor the data stream and extract information from it; this duty can include ^S/^Q flow control, parity checking, and detection of control characters. Control characters may turn into out-of-band information which must be communicated to user space; ^D may become an end-of-file when the application reads to the appropriate point in the data stream, while other characters map onto signals. So the TTY code has to deal with complex signal delivery as well - never a path to a simple code base. Echoing of data - possibly transforming it in the process - must be handled. With the addition of pseudo terminals (PTYs), the TTY code has also become a sort of interprocess communication mechanism, with all of the weird TTY semantics preserved. The TTY code also needs to support networking protocols like PPP without creating performance bottlenecks. 

All told, it's a complicated problem. It is also a problem which seems to interest relatively few developers. The top of `drivers/char/tty_io.c` still reads "Copyright (C) 1991, 1992, Linus Torvalds." Much of the code is still dependent on the big kernel lock. There are deadlocks and race conditions to be found. Almost nobody wants to touch it, but it still mostly works. 

Alan, you are a true wizard :-) The tty layer is one of the very few pieces of kernel code that scares the hell out of me :-)   
\-- [Ingo Molnar](<http://lwn.net/Articles/243968/>), July, 2007 

In recent times, though, an energetic TTY maintainer has stepped forward: Alan Cox. One could almost hear the sighs of relief across the net when this happened; if anybody could clean out that particular set of Augean Stables, it would certainly be Alan, who has the combination of technical skill and attention to detail needed to avoid breaking things. Over the last year, it has been clear that fixing the TTY code has stressed even Alan's skills; the work has been slow and apparently laborious. But it has also been successful at getting the TTY code into better shape while preserving it as a functioning subsystem. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

At least, that was the case until 2.6.31, where the combination of significant changes and some last-minute tweaks led to regressions. Users started to [report](<http://lkml.org/lkml/2009/7/11/125>) that the kdesu application stopped working. The emacs compile mode [started losing output](<http://bugzilla.kernel.org/show_bug.cgi?id=13815>). And so on. It turns out that there were a few separate bugs, not all of which were in the tty layer: 

  * The problem with kdesu appears to be a KDE bug; the application would read too much data, then wonder why the next read didn't have what it wanted. This code worked with the older TTY code, but broke with 2.6.31. There is probably no way to fix it which doesn't saddle the kernel with maintaining weird legacy bug-compatibility code - something the TTY layer does not need more of. 

  * The emacs problem is different. In this case, the compile process would finish its work (writing its final output to the PTY) and exit. Emacs would try to read that final output, but would get a failed read resulting from the SIGCHLD signal sent by the exiting compile process. That failure was unexpected and caused emacs to drop the data. In essence, emacs expected that, by the time the compile process had completed its `close()` of the PTY file descriptor, the data written to that descriptor had been pushed through to the other end and would be available for reading. The 2.6.31 changes broke that assumption. 

The second problem results from the complex nature of TTY data processing. It's not just a serial stream of data; instead, there is the line discipline processing in the middle. In 2.6.31, data written to a PTY will have been queued up for line discipline attention by the time a `close()` is allowed to complete, but there's no assurance that the line discipline code will have actually run and passed the data through to the other end. So the SIGCHLD signal can pass the data and arrive first. 

Alan [thinks this behavior is reasonable](<https://lwn.net/Articles/343845/>); it complies with the applicable standards and can be implemented in a relatively straightforward way. Making a `close()` on a PTY block until the other end has received the data might make emacs work better, but it also risks deadlock if both sides write data and close their file descriptors at the same time. Even so, Alan posted [a "elegant in all the wrong ways" patch](<https://lwn.net/Articles/343847/>) which fixed the problem, but also made it clear that he thought emacs was buggy and that the real fix belonged there. 

Linus merged [a version of this patch](<http://git.kernel.org/linus/3a54297478e6578f96fd54bf4daa1751130aca86>), but he was not happy about it. He [believes](<https://lwn.net/Articles/343849/>) that emacs is correct in its assumptions, and would like to see a better fix which makes the ordering of events clear and deterministic. He [made his frustration clear](<https://lwn.net/Articles/343850/>): 

Why? Why blame emacs? Why call user land buggy, when the bug was introduced by you, and was in the kernel? Why are you fighting it? Why did it take so long to admit that all the regressions were kernel problems? Why were you trying to dismiss the regression report as a user-land bug from the very beginning? 

At that point, it was [Alan's turn](<https://lwn.net/Articles/343851/>) to express frustration; he did not hold back: 

I've been working on fixing it. I have spent a huge amount of time working on the tty stuff trying to gradually get it sane without breaking anything and fixing security holes along the way as they came up. I spent the past two evenings working on the tty regressions. 

However I've had enough. If you think that problem is easy to fix you fix it. Have fun. 

The message included a patch removing Alan as the maintainer of the TTY layer. 

And that is where things stand, as of this writing. The TTY code is unmaintained again, a promising rework has halted partway through, and the person most qualified to fix the problems has thrown up his hands and left the building (though it should be noted that he is participating in the conversation on how the next maintainer, whoever that might be, can fix things). Kernel development will go on, but development in this area will go rather more slowly; the TTY layer has claimed another victim.
