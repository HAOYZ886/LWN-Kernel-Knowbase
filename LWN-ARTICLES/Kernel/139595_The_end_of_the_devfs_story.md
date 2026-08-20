---
title: The end of the devfs story
url: https://lwn.net/Articles/139595/
date: "June 13, 2005"
category: Development model; devfs
author: 
---

Almost one year ago, the kernel developers decided to formally recognize the new development model, where large changes were welcome in the stable 2.6 series. At that time, Greg Kroah-Hartman decided to test out the new model by posting [a patch to remove devfs](<http://lwn.net/Articles/94518/>). The devfs filesystem, a virtual filesystem which provides a dynamic `/dev` directory, had been unpopular with many kernel developers since long before it was [merged in 2.3.46](<http://lwn.net/2000/0217/kernel.php3>). It was never enabled by most distributions, and, in more recent times, had seen little maintenance. Meanwhile, the user-space `udev` utility had developed to the point where it could fill in for devfs. Since there was no 2.7 on the horizon, and 2.6 was officially open to user-visible changes, it seemed like a good time to close the devfs chapter forevermore. 

Except that, as it turns out, the developers were not quite ready to eliminate a user-visible feature on such short notice. After some discussion, it was decided that changes of this kind should happen after a one-year warning period. As a result, a file was created in the Documentation directory (here's the [almost-2.6.12 version](<https://lwn.net/Articles/139599/>)) which listed features scheduled for removal and the target date. Devfs went into the file, with July, 2005 as the time for its ultimate demise. 

July is nearly here, and Greg has not forgotten. He has returned with [a 22-part patch](<https://lwn.net/Articles/139404/>) which removes every trace of devfs from a surprisingly large portion of the kernel. It would seem that devfs had gotten its fingers into just about everything. In the absence of some sort of surprise, this patch seems certain to be merged for 2.6.13. If there are any devfs users out there, they have gotten their last warning.
