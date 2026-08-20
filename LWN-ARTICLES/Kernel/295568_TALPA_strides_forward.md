---
title: TALPA strides forward
url: https://lwn.net/Articles/295568/
date: "August 27, 2008"
category: "Security-Security technologies"
author: "By Jake Edge August 27, 2008"
---

By **Jake Edge**  
August 27, 2008

When last we left TALPA, it was [still floundering around without a solid threat model](<http://lwn.net/Articles/292872/>), but over the last several weeks that part has changed. Eric Paris [proposed](<http://lwn.net/Articles/293835/>) a fairly straightforward--though still somewhat controversial--model for the threats that TALPA is supposed to handle. With that in place, there is at least a framework for kernel hackers to evaluate different ways to solve the problem, while also keeping in mind other potential uses. 

It seems almost tautological, but anti-virus scanning is supposed to, well, scan files. In particular, they scan for known malware and block access to files when they are found to be infected. For better or worse, scanning files is seen as an essential security mechanism by many, so TALPA is trying to provide a means to that end. Paris describes it this way: 

This is a file scanner. There may be all sorts of marketing material or general beliefs that they provide security against all sorts of wide and varied threats (and they do), but in all reality the only threats they provide any help against are those that can be found by scanning files. Simple as that. Some may argue this isn't "good" security and I'm not going to make a strong argument to the contrary, I can stand here for days and show cases where this is highly useful but no one can provide a threat model more than to say, "we attempt to locate files which may be harmful somewhere in the digital ecosystem and try to deny access to that data." 

All of the various scenarios where active processes can infect files with malware or actively try to avoid scanning can be ignored under this model. While this looks like "security theater" to some, it avoids the endless what-ifs that were bogging down earlier discussions. It may not be a threat model that appeals to many of the kernel hackers, but it is one that they can work with. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

To many kernel developers--used to efficiency at nearly any cost--time consuming filesystem scans seem ludicrous, especially since they only "solve" a subset of the malware problem. But the fact remains that Linux users, particularly in "enterprise" environments, believe they need this kind of scanning and are willing to pay for products that provide it. The current methods used by anti-virus vendors to do the scanning are problematic at best, causing users to run kernels tainted with binary modules. With a threat model--however limited--in place, work can proceed to find the right way to add this functionality into the kernel. 

Paris is [narrowing in](<https://lwn.net/Articles/295671/>) on a design that calls out to user space, both synchronously and asynchronously depending on the operation. File access might go something like this: 

  * `open()` \- causes interested user-space programs to be notified asynchronously; anti-virus scanners might kick off a scan if needed
  * `read()/mmap()` \- causes a synchronous user-space notification, which allows anti-virus scanners to block access until scanning is complete; if malware is found, cause the read/mmap to return an error
  * `write()` \- whenever the modification time (mtime) of a file is updated, asynchronously notify user space; this would allow anti-virus scanners to re-scan the data as desired 
  * `close()` \- asynchronous user-space notification; another place where anti-virus scanners could re-scan if the file has been dirtied 

Where and how to store the current scanning status of a file is still an open question. Various proposals have been discussed, starting with a non-persistent flag in the in-memory inode of a file. While simple, that would cause a lot of unnecessary additional scanning as inodes drop out of the cache. Persistent storage of the scanned status of a file alleviates that problem, but runs into another: how do you handle multiple anti-virus products (or, more generally, scanners of various sorts); whose status gets stored with the inode? 

For this reason, user-space scanners will need to keep their own database of information about which inodes have been scanned. For anti-virus scanners, they will also want to keep information about which version of the virus database was used. Depending on the application, that could be stored in extended attributes (xattrs) of the file or in some other application-specific database. In any case, it is not a problem for the kernel, as Ted Ts'o [points out](<https://lwn.net/Articles/295695/>): 

I'm just arguing that there should be absolutely *no* support in the kernel for solving this particular problem, since the question of whether a file has been scanned with a particular version of the virus DB is purely a userspace problem. 

It is important to keep the scanned status out of kernel purview in order to ensure that policy decisions are not handled by the kernel itself. This is in keeping with the longstanding kernel development principle that user space should make all policy decisions. This allows new applications to come along, ones that were perhaps never envisioned when the feature was being designed. For example, Alan Cox [describes](<https://lwn.net/Articles/295696/>) another reason that the state of the file with respect to scanning should be kept in user space: 

This is another application layer matter. At the end of the day why does the kernel care where this data is kept. For all we know someone might want to centralise it or even distribute it between nodes on a clustered file system. 

The latest TALPA design includes an in-memory clean/dirty flag that can short circuit the blocking read notification (when clean). That flag gets set to dirty whenever there is an mtime modification. This optimizes the common case of reading a file that hasn't changed. Further optimizations are possible down the line as Paris mentions: 

If some general xattr namespace is agreed upon for such a thing someday a patch may be acceptable to clear that namespace on mtime update, but I don't plan to do that at this time since comparing the timestamp in the xattr vs mtime should be good enough. 

Various other uses for the kinds of hooks proposed for TALPA have also come up in the discussion. Hierarchical storage management, where data is transparently moved between different kinds of media, might be able to use the blocking read mechanism. File indexing applications and intrusion detection systems could use the mtime change notification as well. This is a perfect example of kernel development in action; after a rough start, the TALPA folks have done a much better job working _with_ the community. 

Some might argue that the kernel development process is somehow suboptimal, but it is the only way to get things into Linux. As the earlier adventures of TALPA would indicate, flouting kernel tradition is likely to go nowhere. While it is still a long way from being included--pesky things like working code are still needed--it is clearly on a path to get there some day, in one form or another.
