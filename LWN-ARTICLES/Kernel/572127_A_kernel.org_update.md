---
title: A kernel.org update
url: https://lwn.net/Articles/572127/
date: "October 29, 2013"
category: "Development tools-Infrastructure; Kernel.org"
author: "By Jonathan Corbet October 29, 2013 2013 Kernel Summit"
---

By **Jonathan Corbet**  
October 29, 2013

* * *

[2013 Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>)

An update on the status of the kernel.org infrastructure is a traditional Kernel Summit feature; the 2013 event upheld that tradition, but with a new speaker. Kernel.org admin Konstantin Ryabitsev started out by saying that he knows nothing about the 2011 security incident; he has deliberately avoided reading the forensics reports that are (apparently) available. His choice is to focus on making kernel.org be as good as it can be now without being driven by past problems. 

He gave a tour of the site's architecture, which your editor will not attempt to reproduce here. In general terms, there is an extensive backend system with a set of machines providing specific services and a large storage array; it is protected by a pair of firewall systems. The front [![\[Konstantin Ryabitsev\]](https://static.lwn.net/images/conf/2013/lce-ks/KonstantinRyabitsev-sm.jpg)](<https://lwn.net/Articles/572143/>) end consists of a pair of servers, each of which runs two virtual machines; one of them handles git and dynamic content, while the other serves static content. 

The front end systems are currently located in Palo Alto, CA and Portland, OR. One will be added in Seoul sometime around the middle of 2014, and another one in Beijing, which will only serve git trees, "soon." Work is also proceeding on the installation of a front end system in Montreal. 

There is an elaborate process for accepting updates from developers and propagating them through the system. This mechanism has been sped up considerably in recent times; code pushed into kernel.org can be generally available in less than a minute. The developers in the session expressed their appreciation of this particular change. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Konstantin was asked about the nearly devastating [git repository corruption problem](<http://jefferai.org/2013/03/29/distillation/>) experienced by the KDE project; what was kernel.org doing to avoid a similar issue? It comes down to using the storage array to take frequent snapshots and to keep them for a long period of time. In the end, the git repository is smaller than one might think (about 30GB), so keeping a lot of backups is a reasonable thing to do. There are also frequent git-fsck runs and other tests done to ensure that the repositories are in good shape. 

With regard to account management, everybody who wants an account must appear in the kernel's web of trust. That means having a key signed by Linus, Ted Ts'o, or Peter Anvin, or by somebody who has such a key. Anybody who has an entry in the kernel MAINTAINERS file will automatically be approved for an account; anybody else must be explicitly approved by one of a small set of developers. 

With regard to security, two-factor authentication is required for administrative access everywhere. All systems are all running SELinux in enforcing mode — an idea which caused some in the audience to shudder. System logs are stored to a ~~write-only~~ write-once medium. There is also an extensive alert system that calls out unusual activity; that leads to kernel.org users getting an occasional email asking about their recent activity on the system. 

Plans for the next year include faster replication through the mirror network and an updated Bugzilla instance. Further out, there are plans for offsite backups, a git mirror in Europe, a new third-party security review, and the phasing out of the bzip2 compression format. 

[Next: [Security practices](<https://lwn.net/Articles/572128/>)].
