---
title: "Filesystem-oriented flags: sad, messy and not going away"
url: https://lwn.net/Articles/815118/
date: "March 16, 2020"
category: System calls
author: "By Jonathan Corbet March 16, 2020"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 16, 2020

Over the last decade, the addition of [a "flags" argument](<https://lwn.net/Articles/585415/>) to all new system calls, even if no flags are actually needed at the outset, has been widely adopted as a best practice. The result has certainly been greater API extensibility, but we have also seen a proliferation of various types of flags for related system calls. For calls related to files and filesystems, in particular, the available flags have reached a point where some calls will need as many as _three_ arguments for them rather than just one. 

One set of filesystem-oriented flags will be familiar to almost anybody who has worked with the Unix system-call API: the `O_` flags supported by calls like `[open()](<http://man7.org/linux/man-pages/man2/openat.2.html>)`. These flags affect how the call operates in a number of ways; `O_CREAT` will cause the named file to be opened if it does not already exist, `O_NOFOLLOW` causes the open to fail if the final component in the name is a symbolic link, `O_NONBLOCK` requests non-blocking operation, and so on. Some of those flags affect the lookup process (`O_NOFOLLOW`, for example) while others, like `O_NONBLOCK`, affect how the file descriptor created by the call will behave. All are part of one flag namespace that is recognized by all of the `open()` family of system calls. 

`open()` is one way to create a new entry in a directory; `[link()](<http://man7.org/linux/man-pages/man2/link.2.html>)` is another. When the time came to add flags to `link()`, the `linkat()` system call was born; this system call also follows the other relatively new pattern of accepting a file descriptor for the directory in which the operation is to be performed. `linkat()` has a separate flag namespace (the "`AT_` flags") with flags like `AT_SYMLINK_FOLLOW`, which is the opposite of `O_NOFOLLOW`. There is also an `AT_SYMLINK_**NO** FOLLOW` that is _not_ recognized by `linkat()`, but which is understood by calls like [`fchmodat()`](<http://man7.org/linux/man-pages/man2/chmod.2.html>) and [`execveat()`](<http://man7.org/linux/man-pages/man2/execveat.2.html>). There are more `AT_` flags, such as `AT_NO_AUTOMOUNT`, supported by the relatively new [`statx()`](<http://man7.org/linux/man-pages/man2/statx.2.html>) system call. 

[Then there is `openat2()`](<https://lwn.net/Articles/796868/>), which is coming with the 5.6 kernel. Rather than having a separate argument for flags, this system call requires a pointer to an `open_how` structure: 

```
struct open_how {
    	__u64 flags;
    	__u64 mode;
    	__u64 resolve;
        };
```

Here, `flags` contains the `O_` flags common to the `open()` family, while `resolve` contains yet another set of flags (the "`RESOLVE_` flags"). These include `RESOLVE_BENEATH` to limit the lookup to files below the provided directory and `RESOLVE_NO_SYMLINKS`, which is kind of like `O_NOFOLLOW` or `AT_SYMLINK_NOFOLLOW` but different: it blocks symbolic-link traversal at all stages of pathname traversal, rather than just for the final component. 

LWN has occasionally covered the ongoing story of the [proposed `fsinfo()` system call](<https://lwn.net/Articles/813172/>), which provides information about mounted filesystems. This new API also includes a structure pointer as one of its parameters: 

```
struct fsinfo_params {
    	__u32	at_flags;
    	__u32	flags;
    	__u32	request;
    	__u32	Nth;
    	__u32	Mth;
    	__u64	__reserved[3];
        };
```

Here, `at_flags` is, as one would expect, a set of `AT_` flags, while `flags` is yet another set of flags specific to this system call. Recently, though, `fsinfo()` author David Howells [noted](<https://lwn.net/ml/linux-fsdevel/96563.1582901612@warthog.procyon.org.uk/>) that he had been told that `RESOLVE_` flags should be used in preference to `AT_` flags in all new system calls, and asked whether the `AT_` flags should be considered deprecated. He followed up with [a patch](<https://lwn.net/ml/linux-fsdevel/3774367.1583430213@warthog.procyon.org.uk/>) marking the `AT_` flags as being deprecated and adding new `RESOLVE_` flags to cover behaviors that can currently only be requested by `AT_` flags. So, for example, he added `RESOLVE_NO_TERMINAL_SYMLINKS` (later renamed `RESOLVE_NO_TRAILING_SYMLINKS`) to request the same semantics as `AT_SYMLINK_NOFOLLOW`. 

Christian Brauner [argued in favor](<https://lwn.net/ml/linux-fsdevel/20200228152427.rv3crd7akwdhta2r@wittgenstein/>) of moving to `RESOLVE_` flags, noting that some of the semantics that are only available via those flags may be of use in settings beyond `openat()`. He did allow, though, that ""we might end up causing more confusion for userspace due to yet another set of flags"" — though others might argue that it's a bit late to worry about that at this point. 

Linus Torvalds, though, [is not a fan](<https://lwn.net/ml/linux-fsdevel/CAHk-=wiEBNFJ0_riJnpuUXTO7+_HByVo-R3pGoB_84qv3LzHxA@mail.gmail.com/>) of the plan to deprecate the `AT_` flags; he noted that software will continue to use flags like `O_NOFOLLOW` or `AT_SYMLINK_NOFOLLOW`, so they can't go away. He [added](<https://lwn.net/ml/linux-fsdevel/CAHk-=wiaL6zznNtCHKg6+MJuCqDxO=yVfms3qR9A0czjKuSSiA@mail.gmail.com/>): 

And yes, the fact that we then have three different user-visible namespaces (O_xyz flags for open(), AT_xyz flags for linkat(), and now RESOLVE_xyz flags for openat2()) is sad and messy. But it's an inherent messiness from just how the world works. We can't get rid of it. 

Adding multiple flags that do the same thing leads to complexity and confusion, he said; one might thus conclude that any such patch is unlikely to make it into the mainline. He later [said](<https://lwn.net/ml/linux-fsdevel/CAHk-=wgu3Wo_xcjXnwski7JZTwQFaMmKD0hoTZ=hqQv3-YojSg@mail.gmail.com/>) that, if `fsinfo()` needs features controlled by both `AT_` and `RESOLVE_` flags, it should accept both; that, along with the flags specific to that system call, adds up to three different sets of flags for one call. One could reasonably conclude that if, for example, `openat2()` were to implement a feature controlled by an `AT_` flag, it would have to accept a third set of flags as well. 

So the situation may indeed be ""sad and messy"", but it doesn't appear that it will be getting any less messy anytime soon. Perhaps one of the messiest aspects of this API is that there is no type checking for any of these flags fields. Nothing but due care prevents a developer from setting a flag in the wrong field. That one may be hard to correct in a backward-compatible way, even if somebody were to be motivated to do it. It is not the biggest mess to be found in our APIs; we'll continue to muddle on with things as they are.
