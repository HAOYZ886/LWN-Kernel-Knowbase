---
title: The trouble with backporting fixes
url: https://lwn.net/Articles/72982/
date: "February 26, 2004"
category: Security
author: 
---

Most Linux distributors, as a matter of standard procedure, do not fix security problems by upgrading their users to the latest version of the affected program. Instead, the specific fix is painstakingly backported to whatever version was originally shipped, and a minimally disruptive (one hopes) update is released. This approach does help protect users from dealing with new issues caused by unplanned software upgrades, but it poses some risks as well. 

Consider, for example, [this notice](<https://lwn.net/Articles/72978/>) sent out to users of Solar Designer's Openwall Linux. On the topic of the recently discovered `mremap()` vulnerability (the second such), it states: 

Luckily, Linux 2.4.23-ow2 and 2.4.24-ow1 are not affected as these patches already included a kernel bug fix which was later determined to be security-critical and needed to avoid this second mremap(2) system call vulnerability. In fact, it's the exact same fix which went into Linux 2.4.25. 

We asked Solar how it was that his patch, which fixed the problem long before it was reported, was not more widely distributed. His response was that he had sent a patch around, but most distributors did not see at the time that the bug had security implications, so they left it out in order to distribute a minimal fix for the first `mremap()` problem. By insisting on a minimal patch, the distributors left their users open to another vulnerability, and forced them to deal with yet another security update shortly thereafter. 

The free software community, in fact, has a long history of bug fixes which, at some later point, turn out to close a security hole. Certain members of the black hat community spend a lot of time digging through changelogs looking for just this sort of problem. Some of them have a true gift for seeing vulnerabilities where the original developers see only bugs. For these people, software changelogs are a roadmap of potentially exploitable bugs known to exist on most deployed Linux systems. 

Few system administrators enjoy being forced to upgrade a package in a hurry. They have learned through hard experience that such upgrades can introduce no end of problems and make a serious dent in their weekend beer-drinking time. In the end, however, we may be forced to face a simple fact: any bug may potentially have security implications. It may be that the Fedora Project has the right idea: when a security hole must be closed, that should be done by upgrading the whole package to the current version. Relatively young software and the new and unknown bugs it is certain to have may turn out to be safer than staying with an older version, which has old and well-documented bugs.
