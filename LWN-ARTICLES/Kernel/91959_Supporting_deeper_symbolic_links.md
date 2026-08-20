---
title: Supporting deeper symbolic links
url: https://lwn.net/Articles/91959/
date: "June 30, 2004"
category: "Filesystems-Symbolic link depth; Symbolic links"
author: 
---

Linux has long limited filename lookups to a maximum of five chained symbolic links. The limit is a useful way of dealing with symbolic link loops, but that is not why it exists. Following symbolic links is an inherently recursive task; once a link has been resolved, the new destination can be another link, which starts the whole process from the beginning. In general, recursion is frowned on in the kernel; the tight limit on kernel stack space argues against allowing any sort of significant call depth at all. The five-link limit was set because, if the limit were higher, the kernel would risk overrunning the kernel stack when following long chains. 

Users do occasionally run into the five-link limit, and, of course, they complain. The limit imposed by Linus is lower than that found on a number of other Unix-like systems. So there has long been some motivation to raise that limit somewhat. 

Alexander Viro has finally done something about it. His approach was to change the behavior of the filesystem `follow_link()` method slightly. This method has traditionally been charged with finding the target of a symbolic link, then calling back into the virtual filesystem code (via `vfs_follow_link()`) to cause the next stage of resolution to happen. In the new scheme of things, the `follow_link()` method is still free to do the whole job, so unmodified filesystems still work. But the preferred technique is for the filesystem code to simply store the file name for the link target in a place where the VFS code can find it and return. The VFS can then make the `vfs_follow_link()` call itself. 

This seems like a small change, but it has an important effect. The filesystem's `follow_link()` method's stack frame is now gone, since it has returned back to the core VFS code. And the core code can use an in-lined version of `vfs_follow_link()`, rather than calling it (with its own stack frame) from the outside. As a result, two fewer stack frames are required for every step in the resolution of the symbolic link. 

Al figures that this change will enable raising the maximum link depth to eight, or even higher (though there is probably little reason to go beyond eight). That change has not yet happened - all of the filesystems will need to be updated and the patch proven stable first. But the initial set of patches has found its way into Linus's BitKeeper tree, so the process is coming near to its conclusion.
