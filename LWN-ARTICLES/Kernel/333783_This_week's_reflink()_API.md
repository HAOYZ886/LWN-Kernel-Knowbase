---
title: "This week's reflink() API"
url: https://lwn.net/Articles/333783/
date: "May 19, 2009"
category: Filesystems; System calls; reflink
author: "By Jonathan Corbet May 19, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 19, 2009

The proposed `reflink()` system call creates an interesting cross between a hard link and a file copy. The end result of a successful `reflink()` call is a new, distinct file - with its own inode - which shares data blocks with the original file. A copy-on-write policy is used, so the two files remain distinct; if one is modified, the changes will not be visible in the other. This call has a number of uses, including fast snapshotting and as a sort of optimized copy operation. But, as was described [in the previous article on `reflink()`](<http://lwn.net/Articles/331808/>), there is some disagreement over how file ownership and security-related metadata should be handled. 

It comes down to the different use cases for this system call. In the "snapshot" case, security information must be preserved; that, in turn, means that `reflink()` can only be used by the owner of the file (or by a process with sufficient capabilities to get around ownership restrictions). On the other hand, those wanting to use `reflink()` as a fast file copy would rather see security information treated like it would be with a file copy; the user creating the reflink must have read access to the original file and ends up owning the new one. 

For a while, it seemed like the reflink-as-copy use case was simply going to be left out in the cold. But then Joel Becker, the author of the `reflink()` patches, [proposed a compromise](<http://lwn.net/Articles/333003/>). If the process calling `reflink()` had ownership or suitable privilege, the snapshot semantics would prevail. Otherwise, read access would be required and a new set of security attributes would be applied. The idea was to try to automatically do the right thing in all situations. 

In the end, though, this approach didn't fly either. From [Andy Lutomirski's objection](<https://lwn.net/Articles/333786/>): 

There are plenty of syscalls that require some privilege and fail if the caller doesn't have it. But I can think of only one syscall that does *something different* depending on who called it: setuid. 

Please search the web and marvel at the disasters caused by setuid's magical caller-dependent behavior (the sendmail bug is probably the most famous). This proposal for reflink is just asking for bugs where an attacker gets some otherwise privileged program to call reflink but to somehow lack the privileges (CAP_CHOWN, selinux rights, or whatever) to copy security attributes, thus exposing a link with the wrong permissions. 

Others agreed that automagically changing behavior depending on caller privilege was not the best way to go. So Joel went back to the drawing board yet another time. On May 15, he came back with [a new proposal](<https://lwn.net/Articles/333789/>). The `reflink()` API would now look like: 

```
int reflink(const char *oldpath, const char *newpath, int preserve);
```

The new `preserve` parameter would be a set of flags allowing the caller to specify which bits of security-oriented information are to be preserved. Anticipated values are: 

  * `REFLINK_ATTR_OWNER`: keep the ownership of the file the same. The caller must either _be_ the owner or have the `CAP_CHOWN` capability. 

  * `REFLINK_ATTR_SECURITY`: preserves the SELinux/SMACK/TOMOYO linux security state. This flag is only valid if `REFLINK_ATTR_OWNER` is also provided. In the absence of `REFLINK_ATTR_SECURITY`, the new link gets a brand-new security state, as if it were any other new file. 

  * `REFLINK_ATTR_MODE`: the discretionary access control permissions bits remain the same; requires ownership or `CAP_FOWNER`. 

  * `REFLINK_ATTR_ACL`: all access control lists are preserved. This only works if `REFLINK_ATTR_MODE` is specified. 

The API would also provide `REFLINK_ATTR_NONE` and `REFLINK_ATTR_ALL`, with the obvious semantics. Importantly, if the caller lacks the requisite credentials to preserve the requested information, the call will simply fail. There will be no magically-changing semantics depending on the caller's capabilities. 

Joel also proposes some new flags to the `ln` command: 

  * `-r` requests that a reflink be made. 
  * `-P` says that the `reflink()` call should use `REFLINK_ATTR_ALL`
  * `-p` (lower case) is like `-P`, except that it will retry with `REFLINK_ATTR_NONE` if the first call fails. 

There were some question as to whether all the flags are necessary; perhaps all that is really needed is "preserve all" or "preserve none." But Joel feels like one might as well add the flexibility, given that the argument is being added to the API anyway, and there doesn't seem to be that much strong sentiment to the contrary. All told, the `reflink()` API would appear to be stabilizing toward something that everybody can agree on. It's probably late for 2.6.31, but this new system call could conceivably be ready for the 2.6.32 development cycle.
