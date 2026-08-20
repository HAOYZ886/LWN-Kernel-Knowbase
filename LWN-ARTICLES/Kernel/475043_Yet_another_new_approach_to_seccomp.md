---
title: Yet another new approach to seccomp
url: https://lwn.net/Articles/475043/
date: "January 11, 2012"
category: "BPF-Security; Security-seccomp"
author: "By Jonathan Corbet January 11, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 11, 2012

Over the years, we have seen a number of attempts to use the seccomp ("secure computing") mechanism to reduce the range of operations available to a given process. The hope is to use such a mechanism as part of a sandboxing solution that would allow (for example) a web browser to run third-party code in a safer manner. Thus far, all of these attempts have gone down in flames; see [Seccomp filters: no clear path](<https://lwn.net/Articles/450291/>) from last May for the most recent episode in this particular story. 

Things have been quiet on the seccomp front recently - until now. Will Drewry, who has been behind the recent attempts to enhance seccomp, has come up with [an interesting new approach](<https://lwn.net/Articles/475019/>) to the problem. Whether this attempt will be more successful than its predecessors remains to be seen, but Will has managed to step around some of the traps that doomed his previous attempt. 

In the last seccomp discussion, there was a fair amount of pressure to adapt the kernel's tracing infrastructure to this task; there was also resistance to using that infrastructure in that way. As explained in detail in the patch posting, Will has come to the conclusion that the tracing infrastructure is not really fit for the task anyway: 

At every turn, it appears that the tracing infrastructure was unsuited for being used for attack surface reduction or as a larger security subsystem on its own. It is well suited for feeding a policy enforcement mechanism (like seccomp), but not for letting the logic co-exist. It doesn't mean that it has security problems, just that there will be a continued struggle between having a really good perf system and and really good kernel attack surface reduction system if they were merged. 

Will's new approach has a stroke of brilliance to it: rather than use the ftrace filter mechanism, he has repurposed the networking layer's packet filtering mechanism (BPF). The BPF code normally operates on packets; in the seccomp context, instead, it operates on the register set at the time of each system call. The registers will contain the system call number and its parameters, allowing the filter to make a wide range of decisions on what will (or will not) be allowed. BPF is also well-maintained and well-optimized code; it even has an in-kernel just-in-time compiler. Some of these advantages are lost because seccomp uses its own BPF interpreter; one assumes that a way could be found to merge the two implementations if the underlying idea looks like it will pass muster. 

As of this writing, there has not really been time for comments on the new patch. It will be interesting to see what the developers think. Meanwhile, those wanting more information should see the patch posting and [the documentation file](<https://lwn.net/Articles/475049/>), which includes a sample program showing how to use the new facility.
