---
title: User and group mount options for ext filesystems
url: https://lwn.net/Articles/497106/
date: "May 16, 2012"
category: "Filesystems-Mounting"
author: "By Jake Edge May 16, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 16, 2012

When transporting files between systems using USB sticks or other removable media, one can run into an annoying problem: the UIDs or GIDs of the files on the media don't match those on the system. In most situations, those kinds of devices have a VFAT filesystem that avoids the problem entirely by not storing UID/GID information. But if a user wants to use a "real" filesystem on the device, one of the ext* family for example, it might be useful to specify the local owner of the files. Ludwig Nussel's [patch set](<https://lwn.net/Articles/496891/>) would do just that for ext2, ext3, and ext4 filesystems. 

The patch comes from some work Nussel did ""years ago"", he said, when [re-introducing it](<https://lwn.net/Articles/497141/>). It simply adds two new mount options for ext filesystems. Following in the footsteps of the VFAT filesystem, the patch would add `uid=` and `gid=` options that would treat all files in the filesystem as being owned by that UID/GID combination. When a filesystem is mounted using these options, files retain their ownership on disk, but they appear to be owned by the specified user and group. Existing files cannot have their ownership changed, but new files will be created with the user and group given at mount time. If a different UID/GID combination is desired for new files--to match the UID/GID on the device for example--they can be added to the mount option: 

```
uid=m:n
        gid=x:y
```

which would make the files appear to be owned by m.x and would create new files as n.y. 

One of the first questions to greet Nussel's patch was about putting the code into specific filesystems, rather than the VFS layer. While the VFS seems like the right place, Ted Ts'o [points out](<https://lwn.net/Articles/497209/>) that there is no easy way to do it all there: 

The problem is that there will need to be at least some support in the individual file system, since there isn't a good place for the VFS to intercept the internal file system iget() function to patch in the override uid/gid values. 

So the question at this point is whether it's cleaner to have the functionality split between the VFS and the file system layers (i.e., with the options parsing and storing the override uid/gid values in the super_block structure) or keeping it all in the file system layer, and accepting the duplication of code across multiple file systems. 

Ts'o leaned toward the first approach in that message, but later [reluctantly accepted](<https://lwn.net/Articles/497222/>) the code duplication. From what he could see, there wasn't enough of a win to put it into the VFS. 

There was a little more discussion when Nussel [resent the patch](<https://lwn.net/Articles/497214/>) on May 10. First off, Jan Kara and Ts'o both wanted to see the patch split into three parts (one for each of ext2, 3, and 4), which Nussel did and posted the next day. But, Roland Eggner and Boaz Harrosh were both concerned about the underlying idea of the patch. Circumventing the access restrictions on the files via a mount option is not a sensible way to address what is, really, an administrative problem, they said. 

Eggner [described](<https://lwn.net/Articles/497215/>) how he "solves" the problem for systems he administers by essentially creating and using a static list of UIDs and GIDs. His position is: ""If UIDs differ on machines FORESEEN for file exchange, this is an administrator error, not a kernel deficit."" Furthermore, exchanging files with unexpected systems requires root privileges, he said, so there is no need for the mount option override. 

Like Eggner, Harrosh is [concerned](<https://lwn.net/Articles/497216/>) about security issues with the proposed change. He also doesn't see anything particularly special about the ext filesystems in terms of removable media, noting that VFAT is the dominant choice. Beyond that, he questions the definition of "removable media", and notes that the problem is common in the NFS world: ""we constantly encounter multiple domain uid/gid views, and it does not mean we blow a hole in POSIX security rules."" 

But Neil Brown [sees](<https://lwn.net/Articles/497217/>) things a little differently. He notes that VFAT suffers from limitations including a 4G file size limit and an inability to handle some special characters in file names. That aside, when someone has physical access to a device, it is essentially "removable" in some sense, so that someone may want to easily access the data: 

[...] if I "own" a filesystem - whether because I hold the physical non-encrypted devices or because I know the encryption key - then I want to be able to leverage that "ownership" to full access rights to the contents of the filesystem. By typing in a key or plugging in a device I want to get full "root" access to the filesystem on the device. Not giving that to me is just getting in my way. 

When users insert a VFAT-formatted USB stick or disk, suitably configured systems will give full access to the user by using the VFAT uid/gid options. Nussel's patches essentially just give that same power for ext-formatted devices. While it could certainly lead to problems, those problems are already latent, as Brown pointed out: 

You cannot prevent data destruction on such devices if you lose physical control, and the only workable data privacy option is encryption. Trying to pretend that file permission bits mean anything is extremely naive. 

While Harrosh is concerned that automounters will start using the options, Brown believes that makes sense for removable devices. In the patch, Nussel mentions that it could be done statically in `/etc/fstab` or be handled dynamically through udev rules. The alternative suggested by Harrosh is that root can mount the device and then `chmod` (or `chown`, presumably) the files appropriately. That seems like a pointless exercise that will just have to be repeated, potentially every time the device is plugged into a new system. Eggner's method is certainly workable, at some level, but makes things more difficult and less "user friendly", Brown said. 

In the end, it is a convenience feature. Anyone with physical access to a unencrypted removable device already has the tools available to read the data on it or to put malware onto it. It's a little hard to see how making it easier for legitimate owners of removable USB storage to access their data somehow opens the floodgates for attackers of various sorts. Those of a malicious bent can find any number of ways (live CD, their own Linux system, ...) to access the device as root if they wish. 

It is unclear how prevalent ext-formatted removable devices are, so there may be an argument against adding the feature on those grounds. On the other hand, making the ext family work better may encourage people to use those filesystems more often for removable media. The patches do duplicate code in the three separate filesystems, but the total number of lines is changed is only around 100 lines for each. Moving some of that into the VFS (like parsing the mount options and storing the flags in the superblock) might reduce that a bit, but it's not much code overall. Administrators who are worried about the feature will be able to avoid it entirely, though they may need to keep an eye on their distribution's udev rules. Given that it brings the same convenience as VFAT to ext-formatted devices, it seems like a feature worth having.
