---
title: "Eliminating Android wrapfs \"hackery\""
url: https://lwn.net/Articles/718640/
date: "April 4, 2017"
category: Android
author: "By Jake Edge April 4, 2017 LSFMM 2017"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
April 4, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

As it has evolved over the years, Android has acquired some hacks in how it handles its filesystems. Ted Ts'o would like to see those hacks eliminated, so he led a session at LSFMM 2017 to look at the problem and see what, if any, upstream-acceptable solution could be found. 

Ts'o started with some history. Early Android devices had SD card slots with a FAT filesystem on them. The Android team tried to kill SD cards for the devices, but were ultimately unsuccessful. Meanwhile, the `/sdcard` mount moved into the `/data` partition and a FUSE filesystem was added to emulate the case-insensitive behavior of FAT. 

In fact, there are three separate FUSE mounts used today to enforce different levels of app privilege: read-only, read-write, or no access to `/data`. Based on the capabilities of the app, `nsenter` is used to put it into the namespace where the filesystem is mounted with the proper access restrictions. But this FUSE-based solution has started to become a performance problem, he said. 

[ ![\[Ted Ts'o\]](https://static.lwn.net/images/2017/lsfmm-tso-sm.jpg) ](<https://lwn.net/Articles/718643/>)

The weird permission regime could be handled with a stackable Linux Security Module (LSM) on top of the SELinux module that is already used, he said. But supporting case insensitivity is harder. That had been [discussed some](<https://lwn.net/Articles/685431/>) at LSFMM 2016, but objections were raised because of UTF-8 case-folding requirements and the need to do a brute-force directory search. 

Ts'o looked at what Android is doing; it is only handling ASCII file names. He wondered if a case-folding feature could be added to the virtual filesystem (VFS) layer. It could just use `strcasecmp()` for comparing file names. 

Al Viro pointed out that using `strcasecmp()` would mean that negative directory entries (dentries), which denote names that were looked up but not found, would not work correctly. Unsuccessful searches for "Makefile" with several case variations would leave negative dentries behind, but then "some joker" creates a file with that name and there would be both positive and negative dentries for the "same" name. He suggested that dropping negative dentries for this case might work; it is, he said, what case-insensitive XFS is doing. 

The performance implication of dropping negative dentries is probably just fine for Android. The current solution uses FUSE, after all, so it is not hard to do better than that, Ts'o said. Case-insensitive filesystems could be added as a kernel configuration option and as a mount option for specific mounts. 

So, Ts'o asked, is that something that could go upstream? Clearly the stacked FUSE approach is not going to cut it, so perhaps some variant of this could? Viro suggested simply doing it in ext4, but Ts'o said that Christoph Hellwig had suggested adding something to the VFS so that all filesystems could use it. Some ideas were thrown around about ways to solve the problem without losing the ability to have negative dentries. No one seemed to come up with a solution there. It seems like it might be possible to get the feature in without negative dentry support, though.
