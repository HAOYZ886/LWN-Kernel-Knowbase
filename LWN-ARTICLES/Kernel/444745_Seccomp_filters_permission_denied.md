---
title: "Seccomp filters: permission denied"
url: https://lwn.net/Articles/444745/
date: "May 25, 2011"
category: "Security-seccomp"
author: "By Jonathan Corbet May 25, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 25, 2011

[Last week's article](<https://lwn.net/Articles/443099/>) on the idea of expanding the "secure computing" facility by integrating it with the perf/ftrace mechanism mentioned the unsurprising fact that the developers of the existing security module mechanism were not entirely enthusiastic about the creation of a new and completely different security framework. Since then, discussion of the patch has continued, and opposition has come from an entirely different direction: the tracing and instrumentation developers. 

Peter Zijlstra started off the new discussion with [a brief note](<https://lwn.net/Articles/444746/>) reading: ""I strongly oppose to the perf core being mixed with any sekurity voodoo (or any other active role for that matter)."" Thomas Gleixner [jumped in](<https://lwn.net/Articles/444747/>) with a more detailed description of his objections. In his view, adding security features to tracepoints will add overhead to the tracing system, make it harder to change things in the future, and generally mix tasks which should not be mixed. It would be better, he said, to keep seccomp as a separate facility which can share the filtering mechanism once a suitable set of internal APIs has been worked out. 

Ingo Molnar, a big supporter of this patch, [is undeterred](<https://lwn.net/Articles/444749/>); his belief is that more strongly integrated mechanisms will create a more powerful and useful tool. Since the security decisions need to be made anyway, he would like to see them made using the existing instrumentation to the highest level possible. That argument does not appear to be carrying the day, though; Peter [replied](<https://lwn.net/Articles/444750/>): 

But face it, you can argue until you're blue in the face, but both tglx and I will NAK any and all patches that extend perf/ftrace beyond the passive observing role. 

As of this writing, that's where things stand. Meanwhile, the expanded secure computing mechanism - which didn't use perf in its original form - will miss this merge window and has no clear path into the mainline. Given that Linus [doesn't like the original idea either](<https://lwn.net/Articles/444751/>), it's not at all clear that this functionality has a real future.
