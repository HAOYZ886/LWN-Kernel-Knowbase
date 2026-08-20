---
title: "Rethinking multi-grain timestamps"
url: https://lwn.net/Articles/946394/
date: "October 9, 2023"
category: Filesystems
author: "By Jonathan Corbet October 9, 2023"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
October 9, 2023

One of the significant features added to the mainline kernel during the 6.6 merge window was multi-grain timestamps, which allow the kernel to selectively store file modification times with higher resolution without hurting performance. Unfortunately, this feature also caused some surprising regressions, and was quickly ushered back out of the kernel as a result. It is instructive to look at how this feature went wrong, and how the developers involved plan to move forward from here. 

Filesystems maintain a number of timestamps to record when each file was modified, had its metadata changed, or was accessed (though access-time updates are often turned off for performance reasons). The resolution of these timestamps is relatively coarse, measured in milliseconds; that is usually good enough for users of that information. In certain cases, though, higher resolution is needed; a prominent case is serving files via NFS. Modern NFS protocols can cache file contents aggressively for performance, but those caches must be discarded when the underlying file is modified. One way of informing clients of modifications is through the modification timestamp, but that only works if the resolution of the timestamp is sufficient to reflect frequent changes. 

In theory, recording timestamps at higher resolutions is straightforward, as long as filesystems have space for the extra data. The strength of higher-resolution data is also a problem, though; a low-resolution timestamp will change relatively infrequently, but a timestamp that changes more often must be written back to the filesystem more often. That can increase I/O rates, especially for filesystems that perform journaling, where each metadata update must go through the journal as well. The cost of increased resolution is significant, which is especially problematic since the higher-resolution data will almost never be used. 

The solution was [multi-grain timestamps](<https://lwn.net/ml/linux-kernel/20230503142037.153531-1-jlayton@kernel.org/>), where higher-resolution timestamps for a file are only recorded if somebody is actually paying attention. Normally, timestamp data is only stored at the current, relatively low resolution, meaning that a lot of metadata updates can be skipped for a file that is being written to frequently. If somebody (a process or the kernel NFS server, for example) queries the modification time for a specific file, though, a normally unused bit in the timestamp field will be set to record the fact that the query took place. The next timestamp update will then be done at high resolution on the theory that the modification times for that file are of active interest. As long as somebody keeps querying the modification time for that file, the kernel will continue to update that time in high resolution. 

That is the functionality that was merged for 6.6. The problem is that this algorithm can give misleading results regarding the relative modification times of two files. Imagine the following sequence of events: 

  1. `file1` is written to. 
  2. The modification time for `file2` is queried. 
  3. `file2` is written to. 
  4. The modification time for `file2` is queried (again). 
  5. `file1` is written again. 
  6. The modification time for `file1` is queried. 

After this sequence, the modification time for `file1`, obtained in step 6 above, should be later than that for `file2` — it was the last file written to, after all. But, since its modification time had not been queried, the modification timestamp will be stored in low-resolution. Meanwhile, since there had been queries regarding `file2` (step 2 in particular), its modification timestamp (set in step 3 and queried in step 4) will use the higher resolution. That can cause `file2` to appear to have been changed _after_ `file1`, contrary to what actually happened. And that, in turn, can confuse programs, like `make`, that are interested in the relative modification times of files. 

Once it became clear that this problem existed, it also became clear that multi-grain timestamps could not be shipped in 6.6 in their initial form. Various options were [considered](<https://lwn.net/ml/linux-kernel/20230920-raser-teehaus-029cafd5a6e4@brauner/>), including hiding the feature behind a mount option or just disabling it for now. In the end, though, as [described](<https://lwn.net/ml/linux-kernel/20230921-umgekehrt-buden-a8718451ef7c@brauner/>) by Christian Brauner, the decision was made to simply revert the feature entirely: 

> While discussing various fixes the decision was to go back to the drawing board and ultimately to explore a solution that involves only exposing such fine-grained timestamps to nfs internally and never to userspace. 
> 
> As there are multiple solutions discussed the honest thing to do here is not to fix this up or disable it but to cleanly revert. 

The feature was duly [reverted](<https://git.kernel.org/linus/b5cbe7c00aa0>) from the mainline for the 6.6-rc3 release. 

The shape of what comes next might be seen in [this series from Jeff Layton](<https://lwn.net/ml/linux-kernel/20230922-ctime-v8-0-45f0c236ede1@kernel.org/>), the author of the multi-grain timestamp work. It begins by adding the underlying machinery back to the kernel so that high-resolution timestamps can be selectively stored as before. Timestamps are carefully truncated before being reported to user space, though, so that the higher resolution is not visible outside of the virtual filesystem layer. That should prevent problems like the one described above. 

The series also contains a change to the XFS filesystem, which is the one that benefits most from higher-resolution timestamps when used in conjunction with NFS (other popular filesystems have implemented "change cookie" support to provide the information that NFS clients need to know when to discard caches). With this change, XFS will use the timestamp information to create its own change cookies for NFS; the higher resolution will ensure that the cookies change when the file contents do. 

Layton indicated that he would like to see these changes merged for the 6.7 release. They have been applied to the virtual filesystem tree, and are currently showing up in linux-next, so chances seem good that it will happen that way. If so, high-resolution timestamps will not be as widely available as originally thought, but there is no real indication that there is a need for that resolution in user space in any case; Linus Torvalds was [somewhat critical](<https://lwn.net/ml/linux-kernel/CAHk-=wjGJEgizkXwSWVCnsGnciCKHHsWg+dkw2XAhM+0Tnd0Jw@mail.gmail.com/>) of the idea that this resolution would be necessary or useful. But the most pressing problem — accurate change information for NFS — will hopefully have been solved at last.
