---
title: After the merge window closed...
url: https://lwn.net/Articles/378903/
date: "March 16, 2010"
category: Development model
author: "By Jonathan Corbet March 16, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 16, 2010

Toward the end of the 2.6.33 development cycle, Linus [suggested](<http://lwn.net/Articles/376029/>) that he might make the next merge window a little shorter than usual. And, indeed, 2.6.34-rc1 came out on March 8, twelve days after the 2.6.33 release. A number of trees got caught out in the cold as a result of that change, and that appears to be a result that suits Linus just fine. 

That said, some trees have been pulled after the -rc1 release. These include the trivial tree, with the usual load of spelling fixes and other small changes. There was a large set of ARM changes, including support for a number of new boards and devices. The memory usage controller got a new threshold feature allowing for finer-grained control of (and information about) memory usage. And so on; all told, nearly 1,000 changes have been merged (as of this writing) since the 2.6.34-rc1 release. 

When [the final SCSI pull request](<https://lwn.net/Articles/378907/>) came along, though, Linus found his moment to draw a line in the sand. Linus, it seems, is [getting a little tired](<https://lwn.net/Articles/378908/>) of what he sees as last-minute behavior from some subsystem maintainers: 

I've told people before. The merge window is for _merging_, not for doing development. If you send me something the last day, then there is no "window" any more. And it is _really_ annoying to have fifty pull requests on the last day. I'm not going to take it any more. 

So, Linus [says](<https://lwn.net/Articles/378909/>), he plans to be even more unpredictable in the future. Evidently determinism in this part of the process leads to behavior he doesn't like, so, in the future, developers won't really be able to know how long the merge window will be. In such an environment, most subsystem maintainers will end up working as if the merge window had been reduced to a single week - an idea which had been discussed and rejected at the 2009 Kernel Summit.
