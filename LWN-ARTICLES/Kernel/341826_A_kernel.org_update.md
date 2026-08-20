---
title: A kernel.org update
url: https://lwn.net/Articles/341826/
date: "July 22, 2009"
category: "Development tools-Infrastructure; Kernel.org"
author: "By Jonathan Corbet July 22, 2009"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 22, 2009

Your editor made a brief visit to the [2009 Linux Symposium](<http://www.linuxsymposium.org/2009/>), held in Montreal for the first time. One of the talks which could be seen during that short time was an update on kernel.org, presented by John Hawley. It was an interesting look into a bit of infrastructure that many of us rely upon, but which we tend to take for granted. 

The "state of the server address" started off with the traditional display of bizarre email sent to kernel.org. Suffice to say, the kernel.org administrators get a lot of strange mail. They also have no qualms about displaying that mail (lightly sanitized) for amusement value. 

The board of kernel.org is currently made up of five people: H. Peter Anvin, Jeff Uphoff, Chris Wright, Kees Cook, and Linus Torvalds. Linus, it is said, never attends the board meetings; John assumes that he's busy doing something related to the kernel. Peter continues to serve as the president of the organization, doing the work required to keep it as a nonprofit corporation in good standing. Much of the rest of the work is done by John, who was hired in September, 2008, to be the first full-time system administrator for kernel.org. He is employed by the Linux Foundation to do this job. 

Over the last year, kernel.org has handled the mirroring a of a number of major distribution releases. They have added two new distributions (Gentoo and Moblin) to the mirror network, and Slackware is being added into the mix now. A number of new wiki instances have been added to [wiki.kernel.org](<http://wiki.kernel.org/>). John says that wikis are easy to create; he encourages relevant projects to ask for a kernel.org wiki if it would be helpful. 

Internally, kernel.org runs on ten "disgustingly nice" machines donated by HP. John was strong in his praise of HP and ISC (which provides the bulk of the considerable bandwidth used by kernel.org); without them, kernel.org would not function the way it does. Beyond ISC, there are a couple of machines hosted at the OSU open source lab and one at Umea University in Sweden. A lengthy process has finally gotten all of these machines upgraded to Fedora 9 - just in time, John noted wryly, for Fedora to end support for that distribution. So another round of upgrades in in the works for the near future. 

Another significant change over the last year is the adoption of [GeoDNS](<http://www.caraytech.com/geodns/>) for the kernel.org domains. GeoDNS enables the DNS server to take the location of the requesting system into account and return the addresses of an appropriate set of servers. So kernel.org users now use local kernel.org mirrors, even if they do not explicitly ask for one using a country-specific host name. 

One upcoming initiative is [archive.kernel.org](<http://archive.kernel.org/>). This site is intended to be a permanent archive for older distribution updates. Should somebody find the urge to, say, install Red Hat Linux 5 on a system, it can be satisfied by a visit to archive.kernel.org. Filling in the archive is a work in progress; a number of older distribution releases seem to have fallen off the net. But, experience shows, many of the older releases will be located over time. 

Another work in progress is "boot.kernel.org". This site is intended to be a repository of network-bootable distributions. The distributor can create a tiny boot image which does little more than setting up the network and downloading the next stage from boot.kernel.org. The idea here is that it will become easy to boot rescue or live CD distributions from the net. Distributions which support network installation can also be hosted on boot.kernel.org. This feature should be ready for a public launch sometime in the near future. 

John closed with more amusing email. But, silliness aside, it seems clear that kernel.org is on a solid foundation. It is supporting our community areas going well beyond the kernel itself, and it looks well set to continue doing so for some time.
