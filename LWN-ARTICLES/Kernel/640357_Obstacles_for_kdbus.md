---
title: Obstacles for kdbus
url: https://lwn.net/Articles/640357/
date: "April 15, 2015"
category: kdbus
author: "By Jonathan Corbet April 15, 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
April 15, 2015

The kdbus patch set, which adds a D-Bus-like messaging facility to the kernel, has been through several rounds of review over the course of the last year. The number of comments has been dropping with each review cycle, and the code seemed like it could be on track for a relatively easy merge into the 4.1 kernel. A closer look, though, reveals that there was some residual unhappiness from the last rounds that was always likely to flare up into active opposition when an attempt to merge kdbus was made. And, indeed, that is exactly what happened when Greg Kroah-Hartman [sent a pull request](<https://lwn.net/Articles/640360/>) to Linus on April 13\. 

This conversation is in full swing as of this writing, so an attempt to fully summarize it would be futile. In brief, though, the complaints take a number of forms. There is unhappiness with the performance of kdbus — a bit surprising, since performance is one of the motivating factors behind this development. There are a number of security-related concerns, especially around how the bus collects and transmits metadata about connected processes. Kdbus is still said to not play well with containers. Some developers find the complexity daunting. And so on. 

The core of the disagreement, arguably, can be found in [this message from Greg](<https://lwn.net/Articles/640362/>). There, he agreed that the design was ""unfortunate"" (though he later retracted that statement), and said that kdbus needed to be taken in its current form even if it is not ideal: 

D-Bus is a specification that has been out there for over a decade, and we are not designing anything new here, but rather implementing it as designed. We have to be compatible to the existing users of the DBus system, and don't have the luxury of being able to change core things like this and expect the world to be able to change just because the design is not as clean as it should/could be. 

Again, just like getting horrid hardware to work properly, sometimes we have to write odd code. Or having to implement a network protocol that doesn't seem to be designed "perfectly", yet is used by a few hundred million systems so we have to remain compatible. This is all that we are doing here for stuff like this. 

Remember, this is called kDBUS, not kGENERICIPC, no matter how much we would have liked that to happen from a kernel standpoint. :) 

It is probably fair to say that those who are opposed to kdbus in its current form would rather that it were, indeed, kGENERICIPC. They seem to feel that it should be able to support what is needed to implement D-Bus efficiently, but the D-Bus-specific parts, perhaps, should go into user space. After all, there are only so many interprocess communication mechanisms that can be merged into the kernel; the one that goes in, many developers think, should be free of known flaws and should be able to do more than reimplement the D-Bus protocol. 

It is hard to say at this point how this discussion will play out or what Linus will decide to do in the end. The chances are good, though, that enough high-profile developers have expressed opposition to derail the merging of kdbus in this development cycle. Complete consensus is not always required to get code into the kernel, but getting code merged when there is serious opposition is still quite hard. This story, it seems, may go on for a while yet.
