---
title: "Crash recovery for user-space block drivers"
url: https://lwn.net/Articles/906097/
date: "August 29, 2022"
category: "Block layer-Block drivers; io uring"
author: "By Jonathan Corbet August 29, 2022"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 29, 2022

A new [user-space block driver mechanism](<https://lwn.net/Articles/903855/>) entered the kernel during the 6.0 merge window. This subsystem, called "ublk", uses [io_uring](<https://lwn.net/Articles/776703/>) to communicate with user-space drivers, resulting in some impressive performance numbers. Ublk has a lot of interesting potential, but the current use cases for it are not entirely clear. The recently posted [crash-recovery mechanism](<https://lwn.net/ml/linux-kernel/20220824054744.77812-1-ZiyangZhang@linux.alibaba.com/>) for ublk makes it clear, though, that those use cases do exist. 

If an in-kernel block driver crashes, it is likely to bring down the entire kernel with it. Putting those drivers into user space can, theoretically, result in a more robust system, since the kernel can now survive a driver crash. With ublk as found in the 6.0 kernel, though, a driver crash will result in the associated devices disappearing and all outstanding I/O requests failing. From a user's point of view, this result may be nearly indistinguishable from a complete crash of the system. As patch author Ziyang Zhang notes in the cover letter, some users might be disappointed by this outcome: 

> This is not a good choice in practice because users do not expect aborted requests, I/O errors and a released device. They may want a recovery mechanism so that no requests are aborted and no I/O error occurs. Anyway, users just want everything works as usual. 

The goal of this patch set is to grant this wish. 

A user-space block driver that implements crash recovery should set up its ublk devices with the new `UBLK_F_USER_RECOVERY` flag. There is also an optional flag, `UBLK_F_USER_RECOVERY_REISSUE`, that controls how recovery is done; more on that below. After setup, no other changes are needed for normal driver operation. 

Should a recovery-capable ublk driver crash, the kernel will stop the associated I/O request queues to prevent the addition of future requests, then wait patiently for a new driver process to come along. That wait can be indefinite; if a driver claims to be able to do recovery, then the kernel will expect it to live up to that claim. There is no notification mechanism for a driver crash; user space is required to notice on its own that the driver has come to an untimely end and start a new one. 

That new driver process will connect to the ublk subsystem and issue the `START_USER_RECOVERY` command. That causes ublk to verify that the old driver is really gone and clean up after it, including dealing with all of the outstanding I/O requests. Any requests that showed up after the crash and were not accepted by the old driver can simply be requeued to the new one. Requests that were accepted may have to be handled a bit more carefully, though, since the kernel does not know if they were actually executed or not. 

There are, evidently, some ublk back-ends that cannot properly deal with duplicated writes; such writes must be avoided in that case. That is what the `UBLK_F_USER_RECOVERY_REISSUE` flag is for; if it is present, all outstanding requests will be reissued. Otherwise, requests that had been picked up by the driver, but for which no completion status had been posted, will fail with an error status. This will happen even with read requests, which one would normally expect to be harmless if repeated. 

After starting the recovery process, the new driver should reconnect to each device and issue a new `FETCH_REQ` command on each to enable the flow of I/O requests. Once all of the devices have been set up, an `END_USER_RECOVERY` command will restart the request queue and get everything moving again. With luck, users may not even notice that the block driver crashed and was replaced. 

The ublk subsystem came out of Red Hat and only includes a simple file-backed driver, essentially replicating the loop driver, as an example. At the time, various use cases for this subsystem were mentioned in a vague way, but it was not clear how (or if) it is being used outside of a demonstration mode. It looks a bit like an interesting solution waiting for a problem. 

The appearance of this recovery mechanism from a different company (Alibaba), just a few weeks after ublk was merged, suggests that more advanced use cases exist, and that ublk is, indeed, already in active use. This sort of recovery mechanism tends not to be developed in the absence of some hard experience indicating that it is necessary. Hopefully some of these real-world use cases will come to light — with code — so that the rest of the world can benefit from this work. 

Just as usefully, this information might give some clues about where Linux is headed in the coming years. The effort to blur the boundaries between kernel tasks and those handled in user space shows no signs of slowing down; it would not be surprising to see more ublk-like mechanisms in the future. It would be interesting indeed to have an idea of where these changes are taking us — and to be shown that it isn't a world where development moves to proprietary, user-space drivers.
