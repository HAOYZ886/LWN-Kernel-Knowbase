---
title: Finding the boundaries for stable kernel patches
url: https://lwn.net/Articles/130555/
date: "April 5, 2005"
category: Development model
author: 
---

Greg Kroah-Hartman [started off the 2.6.11.6 process](<https://lwn.net/Articles/130556/>) in the usual way: a posting of all patches proposed for inclusion in that kernel release. The development community was invited to complain about any patches which do not appear to meet [the criteria](<https://lwn.net/Articles/126915/>) for the extra-stable 2.6 kernels. This time around, somebody complained. 

[The patch in question](<https://lwn.net/Articles/130559/>) is a fix to the BIC TCP congestion control algorithm (congestion avoidance, including BIC, was covered here [two weeks ago](<https://lwn.net/Articles/128681/>)). BIC is supposed to perform a binary search to quickly find the optimal congestion window size. Due to a mistake in the TCP dropped packet code, however, that search was not being performed, and BIC was not working as expected. The (very small) patch makes BIC work the way its designers intended, and would seem to be a useful addition. 

As Ted Ts'o [pointed out](<https://lwn.net/Articles/130560/>), however, the rules for these kernels include: 

It must fix a real bug that bothers people (not a, "This could be a problem..." type thing.) 

It is safe to say that the kernel mailing lists have not been overwhelmed by users complaining that BIC was not converging properly on the best congestion window size. In fact, no users have complained. So, it could be argued, the BIC fix, while worthy, should be merged for 2.6.12 and left out of the 2.6.11.x series. 

[An answer](<https://lwn.net/Articles/130580/>) came from David Miller: 

An incorrect implementation of any congestion control algorithm has ramifications not considered when the congestion control author verified the design of his algorithm. This has a large impact on every user on the internet, not just Linux machines. 

David concluded that, since BIC is enabled by default in the 2.6 kernel, this sort of implementation fix should take a high priority. This view seems likely to prevail for this particular patch. Expect more debates, however, as the kernel developers figure out just where the line should be drawn for patches being considered for inclusion into the stable 2.6 kernels.
