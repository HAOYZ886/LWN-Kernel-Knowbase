---
title: "File-based capabilities"
url: https://lwn.net/Articles/211883/
date: "November 29, 2006"
category: "Capabilities; Security-Security technologies"
author: 
---

The capability model has some real appeal. It replaces the "all or nothing" security model inherent in the root account with a set of fine-grained permissions describing exactly what a given process can do. Linux has supported capabilities for years, but this feature has seen little use for a number of reasons; see [this article from last September](<http://lwn.net/Articles/199004/>) for more general discussion of capabilities. 

The fact that capabilities have not been used much has not stopped developers from trying to improve the feature. The latest attempt is the [file capabilities](<http://lwn.net/Articles/211207/>) patch by Serge Hallyn. This patch allows a system administrator to add specific capabilities to an executable file; when that file is executed, the process's capability masks will be set to the capabilities associated with the file. This feature thus functions somewhat like the file setuid bit, but with finer control. 

On the kernel side, file-based capabilities work through the extended attribute mechanism. Capabilities are added to a file by setting a attribute named `security.capability`; the value of the attribute will be this structure: 

```
struct vfs_cap_data_disk {
    	__le32 version;
    	__le32 effective;
    	__le32 permitted;
    	__le32 inheritable;
        };
```

The `version` field holds the current capability version; the other three hold the expected capability masks. 

There are a few interesting features of this implementation: 

  * One might wonder what keeps the user from just setting an extended attribute and obtaining whatever capabilities might be desired. While setting extended attributes is not a privileged operation, setting attributes whose name starts with "`security.`" is. So, unless the user has root privileges, he or she will not be able to set capability attributes. (For the curious, the other restricted attributes are `trusted.*`, which only root can query or change, and `user.*`, which, in some situations, can only be changed by the owner of the file). 

  * The capability masks stored with the file completely overwrite the process's current capabilities. So, if the root user executes a file with capabilities set, it may run with _fewer_ capabilities than it would have otherwise had. 

  * The setting of capabilities is done outside of the check for filesystems mounted with the `nosuid` option. This behaviour would appear to open the system up to attacks via a removable filesystem created on a different system. 

A set of user-space tools exists for working with file-based capability masks; see [the filesystem capabilities page](<http://www.friedhoff.org/fscaps.html>) for downloads, documentation, and examples. 

Before celebrating the arrival of file capabilities, it is worth asking whether system administrators really need another 31 (at last count) permission bits - multiplied by three separate capability masks - to manage on every executable file on the system. It can be hard to keep file permissions bits in proper order even without capabilities. A full capability-based system would approach SELinux in complexity, and may thus be beyond the ability of most people to manage. But one could use this feature to assign restricted capabilities to programs which currently run setuid root. In many cases, root privilege is only need to bind to a low-numbered socket, adjust the system time, or perform raw I/O. Restricting a program to its needed capabilities should reduce the changes of that program being used to do something unexpected.
