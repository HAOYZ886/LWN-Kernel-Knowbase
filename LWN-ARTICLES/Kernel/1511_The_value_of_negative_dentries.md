---
title: The value of negative dentries
url: https://lwn.net/Articles/1511/
date: "June 4, 2002"
category: Dentry cache
author: 
---

A "directory entry" (dentry) is an internal data structure used to hold the results of looking up a file in the filesystem. The Linux "dentry cache" keeps a number of recently used dentries around; they tend to be useful, since files are often accessed more than once over a short period of time. Finding a file in the dentry cache can save a lot of time by avoiding a full filesystem lookup. 

The kernel also hangs on to "negative dentries," which indicate that the given file does not exist. Andrea Arcangeli recently [noted](<https://lwn.net/Articles/1512/>) that these negative dentries can take up quite a bit of memory, and wondered what possible use they could be. His message included a patch to force negative dentries out of memory quickly. 

It turns out, though, that "this file does not exist" can be useful information. A quick `strace` run on a GNOME application, for example, turns up dozens of lookups on nonexistent files as the application gropes around looking for the unbelievable number of libraries it needs. Similarly, apache is continually looking for `.htaccess` files, shells look for executables, etc. It is more than worthwhile to be able to determine that a file doesn't exist without an expensive filesystem call - especially for file names that are often looked up. So negative dentries will stay. 

There is one optimization that can be made, though. In Andrea's case, the negative dentries were created by deleting a large directory full of files. When a file is deleted, it is relatively unlikely that it will be looked up again soon, and keeping a negative dentry around is less useful. In this case, perhaps, it is better to just forget about the file name altogether.
