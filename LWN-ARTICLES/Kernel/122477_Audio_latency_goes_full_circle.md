---
title: Audio latency goes full circle
url: https://lwn.net/Articles/122477/
date: "February 8, 2005"
category: Latency
author: 
---

[Two weeks ago](<http://lwn.net/Articles/120797/>), it appeared that a solution to the problem of low-latency scheduling for audio applications had been found. Ingo Molnar's approach, which allowed unprivileged processes to use the realtime scheduling modes as long as they did not use more than an administrator-specified portion of the available CPU time, seemed like a reasonably straightforward way to go. Ingo's patch had gone into the -mm tree for further testing. 

The rlimit approach keeps a rogue process from taking over the system entirely. It does not, however, prevent abuse by poorly-behaved software. If even limited access to realtime scheduling became widely available on Linux systems, it would only be a matter of time until developers figured out that they could make their programs seem faster by using the realtime mode. Proprietary applications could be particularly problematic in this regard; distributors would likely rip out unwarranted realtime scheduling calls in free software that they ship, but that cannot be done with proprietary code. 

Other concerns with the rlimit approach include the need for some audio applications to get fast access to the CPU even if they require 100% of the available time, and general unease with tweaking the scheduler for this use. The end result is that the rlimit patch has come back out of -mm, and Ingo has [said](<https://lwn.net/Articles/122478/>): 

i'm not opposed to the LSM solution per se, especially given that none of the other solutions in existence are fully satisfactory (and thus acceptable for the scheduler currently). The LSM patch is clearly the least intrusive solution. 

Those who have been following the discussion will remember that the whole long thing began because certain kernel developers did not feel that the realtime security module (which gives members of an administrator-specified group access to realtime scheduling) was acceptable for inclusion. So the discussion has come back to where it started, and it appears that the realtime security module will be merged (though that had not happened as of this writing). Ingo [apologized](<https://lwn.net/Articles/122479/>) for the whole thing, explaining it this way: 

it is just an unfortunate situation that the issue here is _not_ clear-cut at all. It is a longstanding habit on lkml to try to solve things as cleanly and generally as possible, but there are occasional cases where this is just not possible. 

One remaining problem with the realtime security module is that it gives audio users the right to monopolize the processor with any program they run, not just audio utilities. Making the audio programs run in a setgid mode might seem like a way around that issue, except for the fact that the GTK+ toolkit [actively prevents](<https://lwn.net/Articles/122481/>) things from working that way. The unfortunate result is that users must be given more privilege than they actually need. Most of the time, that should be acceptable; multi-user audio workstations are likely to be relatively rare.
