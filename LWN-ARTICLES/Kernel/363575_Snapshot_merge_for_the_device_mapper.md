---
title: Snapshot merge for the device mapper
url: https://lwn.net/Articles/363575/
date: "November 24, 2009"
category: Device mapper
author: "By Jonathan Corbet November 24, 2009"
---

By **Jonathan Corbet**  
November 24, 2009

Last week, LWN [looked at the use of Btrfs snapshots](<http://lwn.net/Articles/362500/>) to help system administrators recover from problematic upgrades. Btrfs is not the only snapshot mechanism in the kernel, though; the device mapper layer has had this capability for some time. What is missing from DM is the ability to restore the "origin" (main) device to an earlier state if need be. So the device mapper, in its current form, cannot be used to roll back an unfortunate upgrade without taking the system down and copying data. 

That situation could change soon, possibly as early as 2.6.33. Mike Snitzer has posted patches for [a snapshot-merge target](<http://lwn.net/Articles/363203/>) for DM. This target, simply, merges a snapshot back to the origin device, restoring the state of that device to what it was when the snapshot was taken. So a system administrator could snapshot the device immediately prior to an upgrade, then get back to the pre-upgrade state if things do not go well. 

One nice feature is that merging a snapshot preserves the state of all other snapshots on the device. So our system administrator could take another snapshot after the failed upgrade, before returning to the previous state. That post-upgrade snapshot would continue to exist, allowing the cherry-picking of any files with changes that should persist after the system as a whole is rolled back. 

DM maintainer Alasdair Kergon has told your editor that he'll be reviewing this code shortly, and that it may find its way into linux-next in the near future.
