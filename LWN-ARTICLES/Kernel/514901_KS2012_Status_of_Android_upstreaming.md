---
title: "KS2012: Status of Android upstreaming"
url: https://lwn.net/Articles/514901/
date: "September 5, 2012"
category: Android
author: "By Michael Kerrisk September 5, 2012 2012 Kernel Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Michael Kerrisk**  
September 5, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

Anyone who has paid even slight attention to the progress of the mainlining of the Android modifications to the Linux kernel will be aware that the process has had its ups and downs. An initial attempt to mainline the changes via the staging tree [ended in failure when the code was removed in kernel 2.6.33](<https://lwn.net/Articles/372419/>) in late 2010\. Nevertheless, at the 2011 Kernel Summit, [kernel developers indicated a willingness](<https://lwn.net/Articles/464298/>) to mainline code from Android, and [starting with Linux 3.3](<https://lwn.net/Articles/472984/>), various Android pieces were brought back into the staging tree. (On the Android side this was guided by the [Android Mainlining Project](<http://elinux.org/Android_Mainlining_Project>).) The purpose of John Stultz's presentation on day 1 of the 2012 Kernel Summit was to review the current status of upstreaming of the Android code and outline the work yet to be done. 

John began by reviewing the progress in recent kernel releases. Linux 3.3 reintroduced a number of pieces to staging, including [ashmem](<https://lwn.net/Articles/452035/>), [binder](<https://lwn.net/Articles/466304/>), [logger](<https://lwn.net/Articles/473999/>), and the [low-memory killer](<https://lwn.net/Articles/511731/>). With the Linux 3.3 release, it became possible to boot Android on a vanilla kernel. Linux 3.4 added some further pieces to the staging tree and also saw a lot of cleanup of the previously merged code. Subsequent kernels have seen further Android code move to the staging tree, including the [wakeup_source](<https://lwn.net/Articles/479841/>) feature and the [Android Gadget driver](<http://thread.gmane.org/gmane.linux.usb.general/56244>). In addition, some code in the staging tree has been converted to use upstream kernel features; for example, Android's `alarm-dev` feature was converted to use the [alarm timers feature](<https://lwn.net/Articles/429925/>) added to Linux in kernel 3.0. 

As of now (i.e., after the closure of the 3.6 merge window), there still remain some major features to merge, including the [ION memory allocator](<https://lwn.net/Articles/480055/>). In addition, various Android pieces still remain in the staging tree (for example, the low-memory killer, ashmem, binder, and logger), and these need to be reworked (or replaced), so that the equivalent functionality is provided in the mainline kernel. However, one has the impression that these technical issues will all be solved, since there's been a general improvement in relations on both sides of the Android/upstream fence; John noted that these days there is much less friction between the two sides, more Android developers are participating in the Linux community, and the Linux community seems more accepting of Android as a project. Nevertheless, John noted a few things that could still be improved on the Android side. In particular, for many releases, the Android developers provided updated code branches for each kernel release, but in more recent times they have skipped doing this for some kernel releases. 

Following John's presentation, there was relatively little discussion, which is perhaps an indication of the fact that kernel developers are reasonably satisfied with the current status and momentum of Android upstreaming. Matthew Garrett asked if John has any feeling about whether other projects are making use of the upstreamed Android code. In response, John noted that Android code is being used as the default Board Support Package for some projects, such as [Firefox OS](<http://en.wikipedia.org/wiki/Firefox_OS>). He also mentioned that the [volatile ranges](<https://lwn.net/Articles/500382/>) code that he is currently developing has a number of potential uses outside of Android. 

Matthew was also curious to know if is there anything that the Linux kernel developers could do to help make the design process for features that are going into Android more open. Right now, most Android features are developed in-house, but perhaps a more open-developed solution might have satisfied other users' requirements. There was some back and forth as to how practical any other kind of model would be, especially given the focus of vendors on product deadlines; the implicit conclusion was that anything other than the status quo was unlikely. 

Overall, the current status of Android upstreaming is very positive, and certainly rather different from the situation a couple of years ago.
