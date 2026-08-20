---
title: A general caching filesystem
url: https://lwn.net/Articles/100321/
date: "September 1, 2004"
category: "CacheFS; Network filesystems-Caching layer"
author: 
---

Many filesystems operate with a relatively slow backing store. Network filesystems are dependent on a network link and a remote server; obtaining a file from such a filesystem can be significantly slower than getting the file locally. Filesystems using slow local media (such as CDROMs) also tend to be slower than those using fast disks. For this reason, it can be desirable to cache data from these filesystems on a local disk. 

Linux, however, has no mechanism which allows filesystems to perform local disk caching. Or, at least, it didn't have such a mechanism; David Howells's [CacheFS patch](<https://lwn.net/Articles/99597/>) changes that. 

With CacheFS, the system administrator can set aside a partition on a block device for file caching. CacheFS will then present an interface which may be used by other filesystems. There is a basic registration interface, and a fairly elaborate mechanism for assigning an index to each file. Different filesystems will have different ways of creating identifiers for files, so CacheFS tries to impose as little policy as possible and let the filesystem code do what it wants. Finally, of course, there is an interface for caching a page from a file, noting changes, removing pages from the cache, etc. 

CacheFS does not attempt to cache entire files; it must be able to deal with the possibility that somebody will try to work with a file which is bigger than the entire cache. It also does not actually guarantee to cache anything; it must be able to perform its own space management, and things must still function even in the absence of an actual cache device. This should not be an obstacle for most filesystems which, by their nature, must be prepared to deal with the real source for their files in the first place. 

CacheFS is meant to work with other filesystems, rather than being used as a standalone filesystem in its own right. Its partitions must be mounted before use, however, and CacheFS uses the mount point to provide a view into the cached filesystem(s). The administrator can even manually force files out of the cache by simply deleting them from the mounted filesystem. 

Interposing a cache between the user and the real filesystem clearly adds another failure point which could result in lost data. CacheFS addresses this issue by performing journaling on the cache contents. If things come to an abrupt halt, CacheFS will be able to replay any lost operations once everything is up and functioning again. 

The current CacheFS patch is used only by the AFS filesystem, but work is in progress to adapt others as well. NFS, in particular, should benefit greatly from CacheFS, especially when NFSv4 (which is designed to allow local caching) is used. Expect this patch to have a relatively easy journey into the mainstream kernel. For those wanting more information, see [the documentation file](<https://lwn.net/Articles/100330/>) included with the patch.
