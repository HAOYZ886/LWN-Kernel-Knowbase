---
title: "Bypassing linux-next"
url: https://lwn.net/Articles/423777/
date: "January 19, 2011"
category: "Development model-linux-next; linux-next"
author: "By Jonathan Corbet January 19, 2011"
---

By **Jonathan Corbet**  
January 19, 2011

It has been almost three years since the [creation](<https://lwn.net/Articles/268881/>) of the linux-next tree; during that time, it has become an indispensable part of the kernel development process. By the time code is merged into the mainline during the merge window, it has already seen a fair amount of integration and compilation testing in linux-next - and even some actual run testing. That has helped to make the merge window run more smoothly. So it's not surprising that developers are getting increasingly grumpy when code is seen to be circumventing linux-next and creating problems in the mainline. 

We've had a couple of examples of that grumpiness in the 2.6.38 cycle. When Al Viro posted his first VFS pull request, linux-next maintainer Stephen Rothwell [complained](<https://lwn.net/Articles/423780/>) that this was his first sighting of that code, despite the fact that it had apparently been around for a few months. Al is known for pulling together mainline submissions at the last minute, so this sort of thing is not entirely surprising; it remains to be seen whether he can be pushed into changing his ways. 

The other complaint came after the merging of the transparent huge pages patch set, which went in by way of Andrew Morton's -mm tree. Tony Luck, having discovered that the ia64 architecture no longer built in the mainline, [asked](<https://lwn.net/Articles/423781/>): 

Didn't Andrew make some rash promise at kernel summit about stopping eating if "-mm" wasn't included in linux-next by the end of November? Must be getting pretty hungry by now. 

Andrew [responded](<https://lwn.net/Articles/423783/>) that ""It's taking a while - Stephen and I are discussing a plan."" Integrating -mm was always going to be a bit of a challenge; linux-next is supposed to contain code which is ready for merging into the mainline, while -mm can carry under-development code for years. Until that gets worked out, though, memory management developers are going to be in a bit of a difficult position; there is no maintainer tree they can get into which feeds into linux-next. Those developers will need to either get their own trees into linux-next (an easy thing to do) or take the complaints when code which lived in -mm is seen by testers for the first time when it hits the mainline.
