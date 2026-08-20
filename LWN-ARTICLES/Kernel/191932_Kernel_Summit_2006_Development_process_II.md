---
title: "Kernel Summit 2006: Development process II"
url: https://lwn.net/Articles/191932/
date: "July 19, 2006"
category: Development model
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
For the final session of the 2006 Kernel Summit, the developers returned to the development process topic. Was there anything which had been learned over the last two days which called for changes in the process? Linus started off by saying that he was happy with how the process is working, and that most other developers seemed happy as well. It might be nice to shorten the cycle just a little bit - getting the stable 2.6.x releases out a few weeks earlier - but that is about it. 

There was some concern about some patches sitting in the -mm kernel for too long, and about patches not getting enough review in general. Some talk of imposing policies requiring (or at least rewarding) review of other peoples' code was heard, but the idea didn't get very far. Linus noted that often, what is really needed is the experience of having a new body of code in the kernel. Review is good, but it is not the whole solution. 

The interface between the developers and the vendor community has its glitches. The maintainers file contains a fair amount of fiction at this point, with the result that some vendors try to submit code to people who have not worked with the kernel in years. There is also the issue of vendors who are trying to do the right thing and contribute drivers, but who get "Christophed" and are never heard from again. Solutions to this problem are hard to come by, however. 

Linus said that he could see no reason to do a 2.7 kernel anytime soon. Given the sort of changes the kernel has been able to absorb, anything requiring a 2.7 would be so large and disruptive that nobody would actually want to do it. There was some semi-serious talk of changing the numbering scheme - putting out a 3.0 or simply dropping the "2.6" prefix - but it didn't get very far. 

The session was uncontentious because, as Linus put it, "people are happy." So the developers broke up early and moved on to the beer phase of the event.
