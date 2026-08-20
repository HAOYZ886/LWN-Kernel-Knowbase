---
title: A software suspend decision point
url: https://lwn.net/Articles/160380/
date: "November 16, 2005"
category: Software suspend; devkmem
author: 
---

The relative calm which has settled around the software suspend subsystem may be about to come to an end. This part of the kernel, which has never worked to everybody's satisfaction, remains subject to different ideas of how the problem should be solved. 

Pavel Machek's user-space software suspend patch was [covered here in September](<http://lwn.net/Articles/153203/>). Pavel has now posted [a new version of the patch](<http://permalink.gmane.org/gmane.linux.power-management.general/974>) with a request that it be merged for 2.6.16. The user-space approach is, clearly, the way Pavel thinks that software suspend should go. Beyond getting some code out of the kernel, this approach makes a number of add-on features, such as graphical displays, image compression, image encryption, network-based suspend, etc., easier to implement. If you want to hang a big pile of features onto the suspend mechanism, you eventually have to get into user space. 

One of the first [responses](<https://lwn.net/Articles/160381/>) came from Dave Jones, who said: 

Just for info: If this goes in, Red Hat/Fedora kernels will fork swsusp development, as this method just will not work there. 

The main issue is the fact that the user-space approach uses `/dev/kmem` to repopulate memory at resume time. Red Hat and Fedora kernels do not allow memory to be overwritten in this way; there are no other applications which need that capability, with the exception of rootkits. Allowing user space to overwrite arbitrary physical pages is, to Dave, not worth it, no matter how many software suspend features it enables. Says Dave: ""I'll take 'rootkit doesnt work' over 'bells and whistles'."" 

Nigel Cunningham, the author of the [Suspend2](<http://www.suspend2.net/>) patches, also has some thoughts on the matter. He has been busily cleaning up the suspend2 patches with an eye toward making them more palatable for merging into the mainline. It turns out that [Nigel has a set of 225 patches](<https://lwn.net/Articles/160382/>) which he will soon make available. Since few people have seen the new patch set, it's not clear what sort of reception it will get. It can be said, though, that 225 patches is a large pile of code. Anybody trying to get a patch set of that size merged needs to have some fairly convincing arguments in hand. 

At some point, Nigel's code mountain will become available, and some sort of decision will have to be made. Software suspend could be transformed into suspend2, or moved partially to user space. Or it could be left more-or-less as it is now. These are three very distinct choices - especially as nobody wants to see a repeat of the situation where the mainline kernel supported more than one software suspend implementation. With luck, when the dust settles, Linux will have a more featureful and reliable software suspend implementation than it does now. But expect some interesting discussion between now and then.
