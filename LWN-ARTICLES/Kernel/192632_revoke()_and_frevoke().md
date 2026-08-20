---
title: revoke() and frevoke()
url: https://lwn.net/Articles/192632/
date: "July 25, 2006"
category: Forced unmount; revoke
author: 
---

A system call found in some Unix variants is `revoke()`: 

```
int revoke(const char *path);
```

This call exists to disconnect processes from files; when called with a given `path`, it will shut down all open file descriptors which refer to the file found at the end of that path. Its initial purpose was to defeat people writing programs that would sit on a serial port and pretend to be `login`. As soon as `revoke()` was called with the device file corresponding to the serial port, any login spoofer would find itself disconnected from the port and unable to fool anybody. Other potential uses exist as well; consider, for example, disconnecting a process from a file which is preventing the unmounting of a filesystem. 

Linux has never had this system call, but this situation could change before too long; Pekka Enberg has posted [an implementation of `revoke()`](<http://lwn.net/Articles/192107/>) for review. Pekka has also added a second version: 

```
int frevoke(int fd);
```

This version, of course, takes an open file descriptor as its argument. In either case, the calling process must either own the file, or it must be able to override file permissions. So `revoke()` gives a process the ability to yank an open file out from underneath processes owned by other users, as long as that process owns the file in question. 

Getting this operation right can be a little tricky, with the result that the current implementation makes some compromises which may not sit well with other developers. The process, simplified, is this: 

  * The code loops through every process on the system; for each process, it iterates through the open file table looking for file descriptors corresponding to the file being revoked. Every time it finds one, it zeroes out the file descriptor entry (making that descriptor unavailable to its erstwhile owner). The file is not actually closed, however; instead, a list of files to be closed is created for later action. 

All of this will be rather slow, but that should not be a huge problem: `revoke()` is not a performance-critical operation. The memory allocation (to add an entry to the list of files to close) is a bit more problematic; if it fails, `revoke()` will abort partway through, having done an unknown amount of damage without having accomplished its goal. 

  * Once all open file descriptors have been shut down, the files themselves can be closed. So `revoke()` steps through the list it created, closing each open file. 

  * There is one sticky little problem remaining: some processes may have used `mmap()` to map the file into their address spaces. The `revoke()` call clearly has to do something about those memory areas, or it will not have completed the job. So a pass through all of the virtual memory areas associated with the file is required; for each one, the `nopage()` method is set to a special version which returns an error status. 

That change will keep a process from faulting in new pages from the revoked file, but does nothing about the pages which are already part of the process's address space. To fix those, it is necessary to wander through the page tables of each process having mapped the file, clearing out any page table entries referring to pages from that file. 

An alternative approach can be seen in the [forced unmount patch](<http://developer.osdl.org/dev/fumount/kernel2/patches/2.6.13-rc3/2/>) by Tigran Aivazian, which has been touched by a number of other developers over its fairly long history (its comments include a credit for the port to the 2.6 kernel). This patch has a different final goal - being able to unmount a filesystem regardless of any current activity - but it must solve the same problem of revoking access to all files on the target filesystem. Rather than clearing out file descriptors, this patch replaces the underlying `file` structure with a new one from the "badfs" filesystem. After this change, any attempted operations on the file will return `EIO`. Memory mappings are cleared with a direct call to `munmap()`. 

The final form of the patch may well be a combination of the two, providing both forced unmount and `revoke()` functionality. In the process, some of the remaining issues (such as how to perform safe locking without slowing down the highly-optimized `read()` and `write()` paths) will need to be worked out. But there is clearly demand for these features, so this work will probably proceed to eventual inclusion in the mainline.
