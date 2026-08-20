---
title: In brief
url: https://lwn.net/Articles/351543/
date: "September 9, 2009"
category: reflink
author: "By Jonathan Corbet September 9, 2009"
---

By **Jonathan Corbet**  
September 9, 2009

**reflink() for 2.6.32**. Joel Becker's [announcement](<https://lwn.net/Articles/351542/>) of his 2.6.32 ocfs2 merge plans included a mention that the [`reflink()`](<http://lwn.net/Articles/333783/>) system call would be merged alongside the ocfs2 changes. A call to `reflink()` creates a lightweight copy, wherein both files share the same blocks in a copy-on-write mode. The final `reflink()` API looks like this: 

```
int reflink(const char *oldpath, const char *newpath, int preserve);
        int reflinkat(int olddirfd, const char *oldpath,
                      int newdirfd, const char *newpath,
          		  int preserve,  int flags);
```

A call to `reflink()` causes `newpath` to look like a copy of `oldpath`. If `preserve` is `REFLINK_ATTR_PRESERVE`, then the entire security state of `oldpath` will be replicated for the new file; this is a privileged operation. Otherwise (if `preserve` is `REFLINK_ATTR_NONE`), `newpath` will get a new security state as if it were an entirely new file. The `reflinkat()` form adds the ability to supply the starting directories for relative paths and flags like the other `*at()` system calls. For more information, see the documentation file at the top of [the `reflink()` patch](<http://git.kernel.org/?p=linux/kernel/git/jlbec/ocfs2.git;a=commitdiff;h=25767b48501de19421ae86c406b2dd42df37baf3;hp=5e404e9ed1b05cafb044bd46792e50197df805ed>). 

Joel's patch adds `reflink()` support for the ocfs2 filesystem; it's not clear whether other filesystems will get `reflink()` support in 2.6.32 or not. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

**A stable debugfs?**. Recurring linux-kernel arguments tend to focus on vitally important issues - like where debugfs should be mounted. The official word is that it belongs on `/sys/kernel/debug`, but there have been ongoing problems with rogue developers mounting it on unofficial places like `/debug` instead. Greg Kroah-Hartman [defends `/sys/kernel/debug`](<https://lwn.net/Articles/351658/>) by noting that debugfs is for kernel developers only; there's no reason for users to be interested in it. 

Except, of course, that there is. The increasing utility of the ftrace framework is making it more interesting beyond kernel development circles. That led Steven Rostedt to [make a suggestion](<https://lwn.net/Articles/351661/>): 

I think that the tracing system has matured beyond a "debug" level and is being enabled on production systems. Both fedora and debian are now shipping kernels with it enabled. Perhaps we should create another pseudo fs that can be like debugfs but for stable ABIs. A new interface could start out in debugfs, but when it has reached a stable interface, then it could be moved to another location to signal this. 

Steven would like a new virtual filesystem for stable kernel ABIs which is easier to work with than sysfs and which can be mounted in a more typing-friendly location. Responses to the suggestion have been scarce so far; somebody will probably need to post a patch to get a real discussion going. 

**data=guarded**. Chris Mason has posted [a new version of the ext3 data=guarded mode patch](<http://lwn.net/Articles/351444/>). The guarded mode works to ensure that data blocks arrive on disk before any metadata changes which reference those blocks. The goal is to provide the performance benefits of the data=writeback mode while avoiding the potential information disclosure (after a crash) problems with that mode. Chris had mentioned in the past that he would like to merge this code for 2.6.32; the latest posting, though, suggests that some work still needs to be done, so it might not be ready in time.
