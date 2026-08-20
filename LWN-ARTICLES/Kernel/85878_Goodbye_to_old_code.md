---
title: Goodbye to old code
url: https://lwn.net/Articles/85878/
date: "May 19, 2004"
category: "Filesystems-InterMezzo; InterMezzo; PC9800 architecture"
author: 
---

One of the most important tasks in kernel maintenance is not the addition of new code, but removal of old code that is no longer useful. Unused code bloats the kernel and, potentially, becomes a breeding ground for bugs and security problems. Getting that code out of the way helps keep the kernel cruft level down. 

In recent times, the ax has fallen on two subsystems. The first is the [InterMezzo filesystem](<http://www.inter-mezzo.org/>), which has been removed for 2.6.7. InterMezzo is a distributed filesystem from Peter Braam and company with a number of interesting ideas, but, apparently, few users. Maintenance has been lacking, and Mr. Braam finally [agreed](<https://lwn.net/Articles/85881/>) that it should be removed, noting ""In the past 4 years nobody has supported InterMezzo sufficiently for it to become successful."" The [Lustre](<http://lustre.org/>) filesystem, which is Mr. Braam's current project, appears to be headed for greater success. 

[A patch](<https://lwn.net/Articles/85883/>) has been posted which removes support for the PC9800 architecture. There have been a few small objections to this removal, drawing [this response](<https://lwn.net/Articles/85886/>) from Alexander Viro: 

So are you volunteering to maintain the port? Maintainers are MIA; the damn thing doesn't compile; all patches it gets are basically blind ones ("we have that API change, this ought to take care of those drivers and let's hope that possible mistakes will be caught by testers"). Considering the lack of testers (kinda hard to test something that refuses to build), the above actually spells in one word: "bitrot". 

There has been a rather conspicuous shortage of people stepping up to maintain the PC9800 port, so chances are that it will be going away soon.
