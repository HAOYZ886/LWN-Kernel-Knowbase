---
title: Optimizing stable pages
url: https://lwn.net/Articles/528031/
date: "December 5, 2012"
category: Data integrity; Stable pages
author: "By Jonathan Corbet December 5, 2012"
---

By **Jonathan Corbet**  
December 5, 2012

The term "stable pages" refers to the concept that the system should not modify the data in a page of memory while that page is being written out to its backing store. Much of the time, writing new data to in-flight pages is not actively harmful; it just results in the writing of the newer data sooner than might be expected. But sometimes, modification of in-flight pages can create trouble; examples include hardware where data integrity features are in use, higher-level RAID implementations, or filesystem-implemented compression schemes. In those cases, unexpected data modification can cause checksum failures or, possibly, data corruption. 

To avoid these problems, the [stable pages feature](<https://lwn.net/Articles/442355/>) was merged for the 3.0 development cycle. This relatively simple patch set simply ensures that any thread trying to modify an under-writeback page blocks until the pending write operation is complete. This patch set, by Darrick Wong, appeared to solve the problem; by blocking inopportune data modifications, potential problems were avoided and everybody would be happy. 

Except that not everybody was happy. In early 2012, some users started [reporting performance problems](<https://lwn.net/Articles/486311/>) associated with stable pages. In retrospect, such reports are not entirely surprising; any change that causes processes to block and wait for asynchronous events is unlikely to make things go faster. In any case, the reported problems were more severe than anybody expected, with multi-second stalls being observed at times. As a result, some users ([Google, for example](<https://lwn.net/Articles/528032/>)) have added patches to their kernels to disable the feature. The performance costs are too high, and, in the absence of a use case like those described above, there is no real advantage to using stable pages in the first place. 

So now Darrick is back with [a new patch set](<https://lwn.net/Articles/526094/>) aimed at improving this situation. The core idea is simple enough: a new flag (`BDI_CAP_STABLE_WRITES`) is added to the `backing_dev_info` structure used to describe a storage device. If that flag is set, the memory management code will enforce stable pages as is done in current kernels. Without the flag, though, attempts to write a page will not be forced to wait for any current writeback activity. So the flag gives the ability to choose between a slow (but maybe safer) mode or a higher-performance mode. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Much of the discussion around this patch set has focused on just how that flag gets set. One possibility is that the driver for the low-level storage device will turn on stable pages; that can happen, for example, when hardware data integrity features are in use. Filesystem code could also enable stable pages if, for example, it is compressing data transparently as that data is written to disk. Thus far, things work fine: if either the storage device or the filesystem implementation requests stable pages, they will be enforced; otherwise things will run in the faster mode. 

The real question is whether the system administrator should be able to change this setting. Initial versions of the patch gave complete control over stable pages to the user by way of a sysfs attribute, but a number of developers complained about that option. Neil Brown [pointed out](<https://lwn.net/Articles/528035/>) that, if the flag could change at any time, he could never rely on it within the MD RAID code; stable pages that could disappear without warning at any time might as well not exist at all. So there was little disagreement that users should never be able to turn off the stable-pages flag. That left the question of whether they should be able to _enable_ the feature, even if neither the hardware nor the filesystem needs it, presumably because it would make them feel safer somehow. Darrick had left that capability in, [saying](<https://lwn.net/Articles/528036/>): 

I dislike the idea that if a program is dirtying pages that are being written out, then I don't really know whether the disk will write the before or after version. If the power goes out before the inevitable second write, how do you know which version you get? Sure would be nice if I could force on stable writes if I'm feeling paranoid. 

Once again, the prevailing opinion seemed to be that there is no actual value provided to the user in that case, so there is no point in making the flag user-settable in either direction. As a result, subsequent updates from Darrick took that feature out. 

Finally, there was some disagreement over how to handle the ext3 filesystem, which is capable of modifying journal pages during writeback even when stable pages are enabled. Darrick's patch changed the filesystem's behavior in a significant way: if the underlying device indicates that stable pages are needed and the filesystem is to be mounted in the `data=ordered` mode, the filesystem will complain and mount it read-only. The idea was that, now that the kernel could determine that a specific configuration was unsafe, it should refuse to operate in that mode. 

At this point, Neil [returned](<https://lwn.net/Articles/528037/>) to point out that, with this behavior, he would not be able to set the "stable pages required" flag in the MD RAID code. Any system running an ext3 filesystem over an MD volume would break, and he doesn't want to deal with the subsequent bug reports. Neil has requested a variant on the flag whereby the storage level could request stable pages on an optional basis. If stable pages are available, the RAID code can depend on that behavior to avoid copying the data internally. But that code can still work without stable pages (by copying the data, thus stabilizing it) as long as it knows that stable pages are unavailable. 

Thus far, no patches adding that feature have appeared; Darrick did, however, post [a patch set](<https://lwn.net/Articles/528038/>) aimed at simply fixing the ext3 problem. It works by changing the stable page mechanism to not depend on the `PG_writeback` page flag; instead, it uses a new flag called `PG_stable`. That allows the journaling layer to mark its pages as being stable without making them look like writeback pages, solving the problem. Comments from developers have pointed out some issues with the patches, not the least of which is that page flags are in extremely short supply. Using a flag to work around a problem with a single, old filesystem may not survive the review process. 

The end result is that, while the form of the solution to the stable page performance issue is reasonably clear, there are still a few details to be dealt with. There appears to be enough interest in fixing this problem to get something worked out. Needless to say, that will not happen for the 3.8 development cycle, but having something in place for 3.9 looks like a reasonable goal.
