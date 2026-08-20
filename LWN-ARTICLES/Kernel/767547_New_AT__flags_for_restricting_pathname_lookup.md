---
title: New AT_ flags for restricting pathname lookup
url: https://lwn.net/Articles/767547/
date: "October 4, 2018"
category: "Filesystems-Virtual filesystem layer"
author: "By Jonathan Corbet October 4, 2018"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 4, 2018

System calls like `[openat()](<http://man7.org/linux/man-pages/man2/open.2.html>) ` have access to the entire filesystem — or, at least, that part of the filesystem that exists in the current mount namespace and which the caller has the permission to access. There are times, though, when it is desirable to reduce that access, usually for reasons of security; that has proved to be especially true in many container use cases. A new [patch set](<https://lwn.net/ml/linux-kernel/20180929103453.12025-1-cyphar@cyphar.com/>) from Aleksa Sarai has revived an old idea: provide a set of `AT_` flags that can be used to control the scope of a given pathname lookup operation. 

There have been previous attempts at restricting pathname lookup, but none of them have been merged thus far. David Drysdale [posted an `O_BENEATH` option](<https://lwn.net/Articles/619146/>) to `openat()` in 2014 that would require the eventual target to be underneath the starting directory (as provided to `openat()`) in the filesystem hierarchy. More recently, Al Viro suggested `[AT_NO_JUMPS](<https://lwn.net/Articles/723057/>)` as a way of preventing lookups from venturing outside of the current directory hierarchy or the starting directory's mount point. Both ideas have attracted interest, but neither has yet been pushed long or hard enough to make it into the mainline. 

Sarai's venture into this territory takes the form of several new `AT_` flags that can be used with system calls like `openat()`: 

  * `AT_BENEATH` would, similar to `O_BENEATH`, prevent the pathname lookup from moving above the starting point in the filesystem hierarchy. So, as a simple example, an attempt to open `../foo` would be blocked. This option _does_ allow the use of "`..`" in a pathname as long as the result remains below the starting point, though, so opening `foo/../bar` would work. 
  * `AT_XDEV` prevents the lookup from crossing a mount-point boundary in either the upward or downward direction. 
  * `AT_NO_PROCLINK` prevents the following of symbolic links found in the `/proc` hierarchy; in particular, it is aimed at the links found under `fd/` in any specific process's directory. 
  * `AT_NO_SYMLINK` prevents following any symbolic links at all, including those blocked by `AT_NO_PROCLINK`. 
  * `AT_THIS_ROOT` performs the equivalent of a [`chroot()`](<http://man7.org/linux/man-pages/man2/chroot.2.html>) call (to the starting directory) prior to the beginning of pathname lookup. This option, too, is meant to constrain lookups to the given directory hierarchy; it will also change how absolute symbolic links are interpreted. 

There are numerous use cases for these new flags, but the driving force this time around would appear to be container workloads and, in particular, runtime systems for containers. Those systems often have to look inside a container and, perhaps, act on files within a container's directory hierarchy. If the container itself is compromised or otherwise malicious, it can attempt to play games with its filesystems to confuse the runtime system and gain access to the host. 

This posting got a reception that was positive overall, but with a number of concerns about the details. For example, Jann Horn [liked `AT_BENEATH`](<https://lwn.net/ml/linux-kernel/CAG48ez17EQuJQAZUg5hDFXhkjnnVFh39=aD+j0FBdsoTONSGEA@mail.gmail.com/>), but would rather that it forbade the use of "`..`" entirely, even if the result remains beneath the starting point. Doing so would help to block exploitation of various types of directory-traversal bugs, he said. Sarai [responded](<https://lwn.net/ml/linux-kernel/20181001160431.emb6b2hf32b754cl@ryuk/>) that 37% of all the symbolic links on his system contained "`..`"; ""this indicates to me that you would be restricting a large amount of reasonable resolutions because of this restriction"". That said, he indicated a willingness to change the behavior if need be. 

Horn also [complained](<https://lwn.net/ml/linux-kernel/CAG48ez30WJhbsro2HOc_DR7V91M+hNFzBP5ogRMZaxbAORvqzg@mail.gmail.com/>) about the ""footgun potential"" of `AT_THIS_ROOT` which, he said, shares all of the security failings of `chroot()`. He described a scenario where a hostile container could force an escape by moving directories around: ""If the root of your walk is below an attacker-controlled directory, this of course means that you lose instantly"". A possible mitigation here would be to require the starting directory in `AT_THIS_ROOT` lookups to be a mount point; Sarai was [amenable](<https://lwn.net/ml/linux-kernel/20181001054246.gfinmx3api7kjhmc@ryuk/>) to making this change as well. 

Horn, along with [Andy Lutomirski](<https://lwn.net/ml/linux-kernel/F0E08B90-F10B-4897-913D-CA18E99A987D@amacapital.net/>), questioned the container-management use case; as Lutomirski put it: ""Any sane container is based on pivot_root or similar, so the runtime can just do the walk in the container context"". In this particular case, it [turns out](<https://lwn.net/ml/linux-kernel/20181001054246.gfinmx3api7kjhmc@ryuk/>) that part of the problem is the result of the fact that the container runtime in question is written in Go: 

You're right about this -- for C runtimes. In Go we cannot do a raw clone() or fork() (if you do it manually with RawSyscall you'll end with broken runtime state). So you're forced to do fork+exec (which then means that you can't use CLONE_FILES and must use SCM_RIGHTS). Same goes for CLONE_VFORK. 

Since the system cannot use the relatively cheap ways to get into a container's context, it has to use an expensive workaround instead; this expense could be avoided if files could be opened with the new `AT_` flags. Lutomirski [responded](<https://lwn.net/ml/linux-kernel/C89D720F-3CC4-4FA9-9CBB-E41A67360A6B@amacapital.net/>) that he is ""not very sympathetic to the argument that 'Go's runtime model is incompatible with the simpler solution'"". He proposed an alternative that might work in this setting without adding the new flags. 

That alternative might work, but the fact remains that there are other use cases for restricting the scope of pathname lookups; that is why the idea continues to pop up on the kernel's mailing lists. And Lutomirski, too, [agreed](<https://lwn.net/ml/linux-kernel/1EE20CA2-4C8B-4A80-B613-0277D92B376D@amacapital.net/>) that some of the flags seem useful. Whether this implementation will be the one that manages to go all the way to the mainline remains to be seen, but it seems likely that, one of these years, the kernel will gain the ability to control lookups in a way similar to the one that has been proposed here.
