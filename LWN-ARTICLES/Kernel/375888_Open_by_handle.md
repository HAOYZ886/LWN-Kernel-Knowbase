---
title: Open by handle
url: https://lwn.net/Articles/375888/
date: "February 23, 2010"
category: "File handle; Filesystems-Network"
author: "By Jonathan Corbet February 23, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
February 23, 2010

Most Linux users never deal directly with file handles; indeed, most may not even know they exist. Of the rest, the bulk will have an experience limited to the cheery "stale file handle" message seen by NFS users at horribly inopportune times. In fact, a file handle is just a means by which a file can be uniquely identified within a filesystem. Handles are used in NFS, for example, to represent an open file in a way which allows the server to be almost entirely stateless. Handles are not normally used by, or even available to user-space applications. 

Aneesh Kumar is trying to change that situation with [a short patch series](<http://lwn.net/Articles/375566/>) adding two new system calls: 

```
int name_to_handle(const char *name, struct file_handle *handle);
        int open_by_handle(struct file_handle *handle, int flags);
```

The first takes the given `name` and looks up the associated file handle, which is returned in the `handle` structure. That handle can then be passed to `open_by_handle()` to get an open file descriptor for the file. Only privileged users can call `open_by_handle()`; otherwise it could be possible for a malicious local user to bypass the normal permission checks on the directories in the path to a specific file. 

Why would an application developer want to open a file in two steps instead of just calling `open()`? It comes down to the ability to write filesystem servers that run in user space. Such a server could use `name_to_handle()` to generate handles for files on the underlying filesystem; those handles are then passed to the filesystem's clients. At some future time, the client can pass the handle back to actually open the file. This type of feature is also [already used with the XFS filesystem](<https://lwn.net/Articles/375969/>) for backup and restore operations and with a hierarchical storage management system. 

Discussion of these system calls has been minimal, thus far. It does seem that some work will be needed still to better describe what a file handle really is, and, in particular, what its expected lifetime will be. Without some clarity in that area, it will be hard to write applications which can make proper use of file handles.
