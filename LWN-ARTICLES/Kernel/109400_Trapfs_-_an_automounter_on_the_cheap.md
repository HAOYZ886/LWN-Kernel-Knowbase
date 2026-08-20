---
title: "Trapfs - an automounter on the cheap"
url: https://lwn.net/Articles/109400/
date: "November 3, 2004"
category: Automounter; Filesystems; Trapfs
author: 
---

An automounter implements a special filesystem which mounts remote filesystems on demand, when requested by a user-space process. The Linux automounter (autofs) is a mildly complicated subsystem; the [autofsNG patches](<https://lwn.net/Articles/108025/>) make it somewhat more complicated yet. Adam Richter decided that he could make things simpler, and solve a wider class of problems at the same time. The result has been recently posted as [trapfs](<https://lwn.net/Articles/109398/>), a filesystem which can do automounts and more in less than 500 lines. 

Trapfs is derived from ramfs; by itself, it implements a simple, memory-based filesystem. A user-space process can create files, directories, device nodes, etc. in a trapfs filesystem, and everything will work as expected. There is one additional little twist, however: a trapfs filesystem can be mounted with the location of a special helper program given as a parameter. Whenever an attempt is made to look up a nonexistent file, the helper program is invoked and given a chance to cause that file to exist. When the helper exits, trapfs will return whatever the helper left behind to the original caller. 

So, if you want to implement an automounter, you just set up a trapfs filesystem with a little script which can figure out which remote filesystem to mount in response to a lookup request. The task can be done with a screenfull of commands - especially if security is not a big concern. Of course, there are some little details (such as unmounting idle filesystems) which are left as an exercise for the reader, but the basic idea is straightforward. 

Another possibility is to use trapfs to create a devfs-style device filesystem. The helper program responds to lookup requests by seeing if an appropriate device node can be created. 

Whether trapfs will prove useful for real-world tasks remains to be seen. It could have a role, however, in the creation of simple, dynamic filesystems in cases where a more complete solution (using [FUSE](<https://lwn.net/Articles/68104/>), for example) is more work than is justified by the task. Unless there are major objections, Adam plans to try to get trapfs merged in the relatively near future.
