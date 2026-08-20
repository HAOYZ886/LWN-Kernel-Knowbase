---
title: FUSE and private namespaces
url: https://lwn.net/Articles/133848/
date: "April 27, 2005"
category: "Filesystems-In user space; Namespaces-Mount namespaces"
author: 
---

Two weeks ago, we [looked at the opposition to FUSE](<http://lwn.net/Articles/131856/>), or, more specifically, to the strange filesystem semantics it implements. FUSE overrides the VFS permission checking code to establish its own set of rules; the intent is to keep users (even root) from accessing each other's private filesystems. Few people dispute the goal, but the approach that was used failed to please. 

FUSE hacker Miklos Szeredi has tried to address the concerns with [a new patch](<https://lwn.net/Articles/133431/>) implementing "private mounts." The patch creates a new mount flag (`MNT_PRIVATE`); if that flag is set, then only processes belonging to the owner of the mount can see the mounted filesystem at all. To all other processes on the system, these private mounts would be entirely invisible. With this change in place, the permission checking change is no longer needed. 

Unfortunately, nobody likes this idea either. This patch creates a different set of filesystem semantics; in this case, setuid programs run by a user who has private mounts will see a different filesystem than any other process. The filesystem hackers do not wish to see namespaces which change in surprising ways. 

So what _is_ the solution here? Linux does allow for different processes to have different views of the filesystem ("namespaces"). The namespace mechanism could be brought into play to hide FUSE mounts. The problem is that namespaces were never really meant to be shared across the system. A namespace is a process attribute, like the controlling terminal; it is inherited by child processes, but there is no mechanism for passing a namespace to a process which has not inherited it. Users would like to mount their private filesystems and have them available to all of their processes on the system, so having those filesystems in a namespace which is only available to one process tree does not solve the problem. 

As it turns out, there is one way to access namespaces outside of the creating process tree. Jamie Lokier [noticed](<https://lwn.net/Articles/133853/>) that each process's root directory is accessible via `/proc/_pid_ /root`. A new process can be put into another process's namespace simply by setting its root with `chroot()`. If all works as it seems it should, a user-space solution can be envisioned: write a privileged daemon process which can create namespaces and, using file descriptor passing, hand them to interested processes. Those processes can then `chroot()` into that namespace. `chroot()` is a privileged operation, but the code to handle the user side of this operation could be hidden within a PAM module and made completely invisible. 

All that's left is for somebody to actually code this solution. At that point, a glitch or two could come up, but they should be easily fixed with small patches. So there might just be an answer to the FUSE problem after all.
