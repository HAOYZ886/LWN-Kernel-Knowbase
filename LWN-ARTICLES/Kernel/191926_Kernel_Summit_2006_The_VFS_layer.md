---
title: "Kernel Summit 2006: The VFS layer"
url: https://lwn.net/Articles/191926/
date: "July 19, 2006"
category: "Filesystems-Virtual filesystem layer"
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
The session on the virtual filesystem layer was run by Christoph Hellwig and Suparna Bhattacharya. It was an unstructured session which mostly reviewed the list of features that users would like to have. As Christoph noted, the amount of change in the VFS has been relatively small in recent years, an encouraging indication that the code has become reasonably mature. That doesn't stop people from asking for more features, however. 

There is, it seems, an increasing need to move filesystem attributes from the superblock structure into the vfsmount structure. Some of the work which has been done for NFS, along with the increased interest in filesystem namespaces, makes it more likely that filesystems will be mounted in multiple places, possibly with different attributes. 

Better support for user mounts - allowing unprivileged users to mount filesystems - would be nice. Preferably this could be done without requiring that the `mount` utility be installed setuid root, and without setting things up in advance in the `/etc/fstab` file. How that can be made to work in a secure way is yet to be worked out, however. 

Forced unmounts - unmounting a filesystem regardless of any open references to it - is another wishlist item. This, too, is a hard problem. It is somewhat tied to the request for a `revoke()` call, which would detach a process from a file it (formerly) had open. It's all solvable, but making it robust will take some work. 

The "unionfs" idea - mounting multiple filesystems together and seeing the sum of the files and directories in all of them - is an ongoing request. It looks like too much work, however, for a feature which would get little use. So unionfs is not likely to happen anytime soon. The simpler "union mount" idea, where only the top-level directories are merged, is still a possibility, however. 

Ted Ts'o talked briefly about the plans for ext4. He asked if there were any objections to the stated development plan, where a new "ext3dev" filesystem type would be created to receive new features. None were expressed. He did note that doing things this way will have the effect of blocking certain kinds of fixes from getting into ext3. Any fixes which might cause changes in the on-disk format will have to be reserved for ext3dev/ext4 only. 

  * [Next: Scalability](<https://lwn.net/Articles/191929/>).
