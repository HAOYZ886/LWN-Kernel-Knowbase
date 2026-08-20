---
title: "How implementation details become ABI: a case study"
url: https://lwn.net/Articles/614057/
date: "October 1, 2014"
category: "Development model-User-space ABI; renameat2"
author: "By Jonathan Corbet October 1, 2014"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 1, 2014

One of the final changes that went into the mainline kernel repository before the 3.17-rc7 release was [this fix from Mikhail Efremov](<http://git.kernel.org/linus/d2fa4a8476b911782f7e5167db18770222ac40c3>). It affects some low-level code within the virtual filesystem layer that manages name changes in the dentry structure — the structure that handles the mapping between file names and in-kernel inode structures. How that change came to be necessary makes a good lesson in how unintended behaviors can become part of the kernel's ABI over time. 

#### The problem

The addition of the [`renameat2()` system call](<https://lwn.net/Articles/569134/>) in the 3.15 development cycle brought with it a subtle, unintended change in behavior that is best illustrated by example. On a system running (a hopefully updated version of) Bash, one can type a sequence like the following: 

```
$ cd /tmp
        $ touch foo bar
        $ exec 42<bar
        $ ls -l /proc/self/fd/42
        lr-x------ 1 corbet lwn 64 Sep 29 13:01 /proc/self/fd/42 -> /tmp/bar
```

The `exec` command causes the shell to open `bar` as file descriptor 42\. The output of the `ls` shows that, indeed, `bar` is open on that file descriptor. What should happen, though, after the following sequence of commands? 

```
$ mv foo bar
        $ ls -l /proc/self/fd/42
```

On a kernel prior to 3.15, the output will look like this: 

```
lr-x------ 1 corbet lwn 64 Sep 29 13:01 /proc/self/fd/42 -> /tmp/bar (deleted)
```

On later kernels, instead, the result is: 

```
lr-x------ 1 corbet lwn 64 Sep 29 15:00 /proc/self/fd/42 -> /tmp/foo (deleted)
```

When a file with open descriptors is deleted, the actual file remains in existence until all of those file descriptors are closed. On kernels prior to 3.15, the name associated with that deleted file will be the name it had when it was deleted. In newer kernels, instead, a file may, if it is deleted via a rename operation, end up appearing to have the name of the file that was renamed on top of it. 

This change may appear to be nearly irrelevant; who is going to care about the apparent name of a deleted file that is no longer accessible via the filesystem? But it seems that there are scripts out there that do care. One case was outlined by Mikhail in his patch posting: the package update utility on ALT Linux will replace (via a rename) the executable for a running daemon process, then try to find existing processes that are running the older version of the executable. But renaming the new version of the executable on top of the previous one causes any process running the old version to appear to be running something else, so the upgrade process fails. Piotr Karbowski, who appears to have been the first to [report](<https://lkml.org/lkml/2014/9/6/120>) the bug, stated that it made his system unusable. This behavior change did, in fact, cause systems to break. 

#### The cause

Understanding this bug requires delving into `struct dentry` and a somewhat obscure function called `switch_names()` that handles rename operations. The `dentry` structure, since it is charged with name mapping, must contain the file name of interest. But that name can be stored in two different ways. If the length of the name is less than `DNAME_INLINE_LEN` (a value between 32 and 40, chosen for optimal structure alignment), that name will be stored within the dentry structure itself. Otherwise, the `d_name` field will contain a pointer to an externally-allocated string. 

The `switch_names()` function is defined like this: 

```
void switch_names(struct dentry *dentry, struct dentry *target);
```

Its job is to cause `dentry` to have the name currently associated with `target`. When moving names around, `switch_names()` must clearly pay attention to whether internal or external names are being used. Since there are two `dentry` structures to work with, there are four possible combinations. If both names are allocated externally, life is easy: 

```
swap(target->d_name.name, dentry->d_name.name);
```

One might wonder why the two names are exchanged in this way, since the stated purpose is only to affect `dentry`. The swap is done because `target` is about to disappear anyway, so its "name" is not really seen to matter anymore. Swapping allows this code to (1) avoid memory allocations, which, given how deep it is running within the VFS layer, is useful, and (2) not worry about freeing the old name associated with `dentry`, since that will now happen when `target` is freed anyway. 

If, instead, both names are internal, memory allocations are not a concern. Prior to 3.15, the code for that case looked like this: 

```
memcpy(dentry->d_iname, target->d_name.name, target->d_name.len + 1);
        dentry->d_name.len = target->d_name.len;
```

This operation would leave both `dentry` structures appearing to have the same name — different behavior than that seen with the external-names case. Again, since `target` is expected to be destroyed soon, that difference should not really matter. It did start to matter, though, when the cross-rename feature (allowing the names of two files to be atomically swapped) was added in 3.15. In that case, the two names _should_ be switched, as was done in the external case. So, in current kernels, the code looks like this: 

```
unsigned int i;
    
        for (i = 0; i < DNAME_INLINE_LEN / sizeof(long); i++) {
    	swap(((long *) &dentry->d_iname)[i], ((long *) &target->d_iname)[i]);
        }
```

This funny-looking loop allows the swapping of the two names without the need for temporary variables or extra copying. (For completeness, the mixed internal/external cases swap the names). 

As far as everybody could tell, the above code was correct; it made the internal-name case behave like the external-name case did. But if (1) one file is renamed on top of another, and (2) both files have short names, the user-visible behavior of the system changes, and that change caused programs to break. Breaking things in that way goes against one of the fundamental rules of kernel development, so some sort of fix needs to be put into place. 

#### The fix

[Mikhail's original patch](<https://lwn.net/Articles/614063/>) added a flag ("`exchange`") to `switch_names()`. If that flag is set (as would be the case for an atomic file swap operation), the 3.15 behavior holds; otherwise, the code would revert back to the previous behavior for the both-internal case (names would still be swapped in the other cases). This patch was initially rejected; Linus [called it](<https://lwn.net/Articles/614064/>) ""too ugly to live"": 

Yes, we had that hack before, but we didn't make it conditional. It historically was more of a "it's easier to just memcpy the name" than switch things around. Then that became accidental semantics, and that's all normal. But then when we make this explicit and intentional, I really think we should do it *right*, and either switch() the names around or just copy it. 

Having a "switch_names()" function that *neither* switches *nor* copies, and giving it an argument to decide which, but not even do it *right*? That's just too f*cking disgusting for words. 

A proper solution would, thus, cause the "just copy the name to the new dentry" behavior to happen on rename operations in all cases where an explicit swap has not been requested, even those which were not handled that way in the past. Implementing that behavior runs into a problem, though: in the case where both names are external, it may be necessary to allocate memory for a copy of the target file name. Such an allocation would have to be handled in atomic context and would slow down code that needs to run quickly. So a simple solution is not readily available. 

Thus, the developers decided to merge a version of Mikhail's patch for 3.17, even if they don't like it. The patch has changed a bit since Al Viro took the opportunity to clean up the surrounding code a bit. But that code will probably not last beyond the 3.17 release. 

What is likely to happen, instead, is a variant of [this patch](<https://lwn.net/Articles/614066/>) from Al. It adds a reference count to external names, allowing those names to be "copied" by just incrementing the count. Actually freeing the name must be done conditionally based on the results of a decrement-and-test operation. There are some additional complications; the name may be accessed under read-copy-update (RCU) rules, for example, so the actual freeing must happen in an RCU callback. But the idea is simple enough and, since few places actually manipulate the names in `dentry` structures, the implementation is relatively small. 

Still, that is a larger change than anybody would like to see go into 3.17 at this point in the development cycle. So reference-counted external names in `dentry` structures will have to wait until 3.18. Meanwhile, Mikhail's fix has gone in for 3.17 and been marked for the stable updates, so the old behavior will return in the near future. This behavior was accidental and never documented, and the kernel developers seemingly believe that any code relying on it was poorly written to begin with. But, all of that notwithstanding, that behavior has become a part of the kernel ABI, so those developers will preserve it even if they don't like it.
