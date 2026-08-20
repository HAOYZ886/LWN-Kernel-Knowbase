---
title: "LSFMM: Error returns"
url: https://lwn.net/Articles/548353/
date: "May 1, 2013"
category: SCSI
author: "By Jake Edge May 1, 2013 LSFMM Summit 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 1, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Filesystem handling of errors returned from the SCSI subsystem was the topic of the final combined session for the Storage and Filesystem tracks at the 2013 LSFMM Summit. It is a follow-on to an [earlier session](<https://lwn.net/Articles/548295/>) that got a little side-tracked. The discussion was led by Hannes Reinecke and James Bottomley. 

Reinecke displayed the code from `__scsi_error_from_host_byte()` to show attendees the existing errors that can be returned from the SCSI layer. He asked Dave Chinner, who had earlier expressed interest in seeing errors other than `EIO` be returned, how he would like to see those change. Chinner responded that it was more than just what the error code is; filesystem developers needed to understand what the actual failure is. Most important, from his perspective, is whether it could be a temporary failure or not. 

Chinner gave the example of a Fibre Channel link failure, which can take up to 240 seconds to come back. After 30 seconds, though, the filesystem has received an `EIO` and has shut down. If the Fibre Channel comes back in a few minutes, the filesystem has done the wrong thing from the user's perspective. In that case, the filesystem could have tried again in a few minutes and only shut down after getting 3-4 failures in a row. Overall, filesystems are looking for "classes of errors" that will help the higher levels determine whether to retry or give up, he said. 

But, as Martin Petersen pointed out, the storage does not know if the error is temporary. Chinner noted that there are certain types, like read failures, that are often temporary. Alasdair Kergon was concerned about which level should make that kind of decision. If the kernel developers are not careful, every layer and filesystem will repeat the same kind of logic to make those decisions. 

Five or six distinct error codes, one for each class of error, is what Chinner would like to see. He suggested sticking to POSIX error codes (rather than a new set established between storage and filesystem developers) in case they end up leaking to user space. He reiterated that the filesystem developers don't particularly care what the error code is, but they want to know if the error is transient. He suggested something like "transient read error", "permanent read error", and the same for writes, but Petersen noted, again, that no one really knows if the errors are transient. 

There are things a filesystem can do if it knows that there was some kind of media error on a write, Ric Wheeler said. The data could be written to another disk, or written in place elsewhere on the disk. But that can only happen if the filesystem gets some indication other than `EIO`. 

If the errors returned are to change, Chinner asked, what can be done to document those changes? Documentation in the kernel tree is fine, but there needs to be more widespread notification. In the end, it was suggested that posting the information to the fs-devel mailing list (and also to LWN) is the right way to go about it. While some changes were agreed upon in the meeting, it's not clear how and when (or even if) those exact error return values will be used--though some changes are likely. 

[ Thanks to Elena Zannoni, whose detailed notes were a nice supplement to my own. ]
