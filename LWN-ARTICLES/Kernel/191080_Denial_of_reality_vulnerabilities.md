---
title: Denial of reality vulnerabilities
url: https://lwn.net/Articles/191080/
date: "July 12, 2006"
category: "Security-Vulnerabilities"
author: 
---

On July 7, the folks at rPath sent out [a security update](<http://lwn.net/Alerts/190770/>) for a pair of kernel vulnerabilities. The update reads, in part: 

Previous versions of the kernel package are vulnerable to two denial of service attacks. The first allows any local user to fill up file systems by causing core dumps to write to directories to which they do not have write access permissions. 

The bug in question is designated CVE-2006-2451; it was fixed in the [2.6.17.4](<http://lwn.net/Articles/190420/>) kernel release. All kernels since 2.6.13 are vulnerable, but one cannot just rely on the nominal version number: Red Hat helpfully backported this bug into the 2.6.9 kernel shipped with RHEL4. 

Reading the description above, some system administrators may feel that there is no particular urgency in applying this update. The risk that a rogue user would fill up a disk with core dump files may seem small, so an update fixing the problem - and which requires a system reboot to be effective - can maybe be deferred for a while. After all, the Linux kernel core dump code takes pains to avoid overwriting files with core dumps, so the real potential for harm is small. It's a denial of service bug. 

Except that it's not. All that is required is to create a program containing a string in the format understood by `cron`, send it over to `/etc/cron.d`, and use the bug to create a core dump there. Eventually `cron` will wander along, helpfully pick the line it understands out of the surrounding binary junk, and execute (as root) the commands found there. It is a simple and straightforward local root exploit; an example implementation has been posted to the full-disclosure list. 

Paul Starzetz has posted [a complaint](<https://lwn.net/Articles/191089/>) about the characterization of a fully-exploitable vulnerability as a denial of service problem; he has seen this done with other vulnerabilities as well. He is right. "Denial of service" makes the vulnerability seem less severe, especially if it is only exploitable locally. Those words may cause vulnerabilities to remain open longer by inspiring inaction on both the administrator and distributor sides. If a bug can be exploited for privilege escalation, it should not be described as a denial of service problem. 

To its credit, Red Hat (which is where the bug was discovered) notes that the bug could be exploited to gain root privileges. Ubuntu, which [closed the vulnerability](<http://lwn.net/Alerts/190954/>) four days later, says ""This could be exploited to drain available disk space on system partitions, or, under some circumstances, to execute arbitrary code with full root privileges."" This advisory could use an edit as well: "under some circumstances" makes the exploit seem unlikely or difficult. A more accurate wording would be "if the attacker wants." 

Lest it seem that rPath and Ubuntu are receiving too much grief: as of this writing, five days after disclosure, rPath, Ubuntu, and Red Hat are the _only_ distributors to have fixed this problem. They have done the most important part: making an update available. All other distributors who have shipped kernels based on 2.6.13 or later remain vulnerable to a trivial local root exploit. Might this slow response be caused, in part, by the perception that this is a mere local denial of service bug? 

As a community, we feel that we have the best security support out there. Vulnerabilities are not hidden, and fixes come promptly. In cases like this one, however, we have let our users down. Presenting an easily exploitable root vulnerability as a denial of service problem is just the sort of obfuscation that we normally try to avoid. And the fact that a number of distributions remain vulnerable is a failure to live up to our own promises. We can - and must - do better than that.
