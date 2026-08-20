---
title: "Kernel testing and regressions: an example"
url: https://lwn.net/Articles/145008/
date: "July 26, 2005"
category: "Development model-Regressions; Regression testing"
author: 
---

Kernel testing, or the lack thereof, is considered to be a significant part of the kernel quality problem. Recent kernels, while quite good in many regards, contain more bugs than they should because people have not gotten around to testing them before the final release. Many regressions are in device drivers, which present special testing problems: drivers can only be tested by people who have the relevant hardware. Core kernel code, however, is hardware independent and should be easier to test. But bugs can slip through in that code as well. 

Consider, for example, the realtime rlimits feature, which can be used to enable otherwise unprivileged users to run processes with elevated priority. Andreas Steinmetz recently [noticed](<https://lwn.net/Articles/145011/>) that this feature does not work in the 2.6.13-rc3 kernel. This would seem to be just the sort of feedback the process needs: a user, testing a feature in a -rc kernel, found a bug and provided a patch to fix it. As a result, that particular bug will not be present in 2.6.13. 

The only problem is that, as [confirmed](<https://lwn.net/Articles/145012/>) by Ingo Molnar, the bug is a little older than that. In fact, the realtime resource limit feature does not work at all in the stable 2.6.12 kernel, and nobody noticed until now. This is a feature which can be tested by just about anybody, but that work clearly had not been done. Given that nobody appears to be using this feature, Ingo [is not confident](<https://lwn.net/Articles/145017/>) that the fix can go into a 2.6.12 stable release; this one will have to wait for 2.6.13. 

It should be said that testing realtime resource limits is not an entirely straightforward operation; setting that limit requires changes to the PAM library, C library, and the shells as well. Very few distributions - and no major ones - are shipping those changes at this time. Even so, unprivileged realtime scheduling is a feature that a number of people had been asking for. It is a little surprising that none of those people noticed that it failed to work in a major kernel release. Getting comprehensive testing coverage for the kernel is clearly still a problem - even before drivers are taken into account.
