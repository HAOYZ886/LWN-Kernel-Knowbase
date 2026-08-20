---
title: LogFS returns
url: https://lwn.net/Articles/363572/
date: "November 24, 2009"
category: LogFS
author: "By Jonathan Corbet November 24, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 24, 2009

LogFS is a longstanding project by Jorn Engel to create a filesystem for contemporary solid-state storage devices; it was last [covered here](<http://lwn.net/Articles/234441/>) in May, 2007. Since then, LogFS has mostly disappeared from view. As of November 20, though, [LogFS is back](<http://lwn.net/Articles/363198/>) and, seemingly, ready for a mainline merge. Jorn says: 

Logfs has been around a couple of times. Linus last word was "go and don't come back until all format changes are done". Or something along those lines at least. Format changes are done. And I don't even intend to break git-bisect for anyone crazy enough to use logfs for /. 

Sufficiently crazy users seem to be relatively scarce so far. But having more options for upcoming hardware can only be a good thing; it will be interesting to see what results come out as people start to play with this new filesystem.
