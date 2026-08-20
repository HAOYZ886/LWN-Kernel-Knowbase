---
title: trusted_for() bounces off the merge window
url: https://lwn.net/Articles/890957/
date: "April 12, 2022"
category: "Filesystems-Virtual filesystem layer; System calls"
author: "By Jake Edge April 12, 2022"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
April 12, 2022

When last we [looked](<https://lwn.net/Articles/832959/>) in on the proposed `trusted_for()` system call, which would allow user-space interpreters and other tools to ask the kernel whether a file is "trusted" for execution, it looked like it was on-track for the mainline. That was back in October 2020; the patch has been updated multiple times since then, made its way into linux-next, and a [pull request](<https://lwn.net/ml/linux-kernel/20220321161557.495388-1-mic@digikod.net/>) was made by Mickaël Salaün for the 5.18 merge window. But it seems that there will be more to the story of getting this functionality into the kernel, as Linus Torvalds declined to pull `trusted_for()`, at least partly because he did not like the name, but there were other reasons as well. While he is not opposed to the functionality it would provide, he also had strong feelings that a new system call was not the right approach. 

#### Background

The patch has been through 18 versions since it was first [introduced](<https://lwn.net/ml/linux-kernel/20181212081712.32347-1-mic%40digikod.net/>) in 2018. It [started out](<https://lwn.net/Articles/820000/>) as a new flag (`O_MAYEXEC`) for the [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>) system call. The idea behind it is fairly straightforward: the kernel enforces a number of security checks on files before they can be executed, but various kinds of tools can simply read files in order to execute them. Those files are not subject to the same checks, since the kernel is unaware that they contain code to be executed; finding a way to apply the same checks to files that are, effectively, being opened for execution, is the goal of Salaün's work. 

Obviously, user space needs to be involved since the kernel cannot know that any file being opened is going to be used that way—the vast majority of files are not, after all. Python and other tools are interested in supporting security checks for files containing code (see [PEP 578](<https://peps.python.org/pep-0578/>), for example), but there will clearly be a long tail of tools needing to inform the kernel of their intention and some may well resist or be uninterested in doing so. There would be value in having the feature for some types of locked-down systems that only have "well-behaved" tools that make the check. 

Along the way, Al Viro, maintainer of the virtual filesystem (VFS) layer, [complained](<https://lwn.net/ml/linux-kernel/20200810230521.GG1236603%40ZenIV.linux.org.uk/>) that `openat2()` was not the proper place for handling this kind of check. He suggested a new system call, instead. The [next version of the patches](<https://lwn.net/ml/linux-kernel/20200908075956.1069018-1-mic%40digikod.net/>) moved to an `AT_INTERPRETED` flag for the [`faccessat2()`](<https://www.man7.org/linux/man-pages/man2/faccessat.2.html>) system call instead, but Viro [thought](<https://lwn.net/ml/linux-kernel/20200909171316.GW1236603@ZenIV.linux.org.uk/>) that was not any better and again suggested a new system call. 

After a round of bikeshedding about the name, Salaün [decided](<https://lwn.net/ml/linux-kernel/20200924153228.387737-1-mic@digikod.net/>) on `trusted_for()`. The subsequent revisions were mostly cosmetic changes or updating the code for more recent kernels. It looks nearly the same as it did in our article a year and a half ago: 

```
int trusted_for(const int fd, const int usage, const unsigned int flags);
```

The call will check the file indicated by `fd` to see if it is allowed for the `usage` (`TRUSTED_FOR_EXECUTION` is the only option currently defined); `flags` is, as yet, unused. It will return zero if the file is trusted or `EACCESS` if it is not. By default, however, `trusted_for()` does not actually do anything, but there is a new `fs.trusted_for_policy` sysctl knob that can be set to have it check for files on a filesystem mounted with `noexec`, files that do not have execute permission, or both. 

#### No merge

After the 5.18 merge window had closed without `trusted_for()` being pulled, both [Salaün](<https://lwn.net/ml/linux-kernel/f6cf4112-b7b9-7ad7-dbb0-27304176146f@digikod.net/>) and [Kees Cook](<https://lwn.net/ml/linux-kernel/202204041130.F649632@keescook/>) asked about the status. It turns out that Torvalds [was not happy](<https://lwn.net/ml/linux-kernel/CAHk-=wgoC76v-4s0xVr1Xvnx-8xZ8M+LWgyq5qGLA5UBimEXtQ@mail.gmail.com/>) to see a new, non-standard system call with a ""completely random interface with no semantics except for random 'future flags'"". Salaün [disagreed](<https://lwn.net/ml/linux-kernel/816667d8-2a6c-6334-94a4-6127699d4144@digikod.net/>) that the semantics were unspecified; ""I think the semantic is well defined: 'This new syscall enables user space to ask the kernel: is this file descriptor's content trusted to be used for this purpose?'"" 

Torvalds had a few other complaints as well: 

> What the system call seems to actually *want* is basically a new flag to access() (and faccessat()). One that is very close to what X_OK already is. 
> 
> [...] No way will this ever get merged, and whoever came up with that disgusting "trusted_for()" (for WHAT? WHO TRUSTS? WHY?) should look themselves in the mirror. 
> 
> If you add a new X_OK variant to access(), maybe that could fly. 

The `X_OK` flag for `access()` (and `faccessat2()`) is used to determine whether the process has permission to execute a given file, using the real user and group IDs (rather than the effective IDs, which could be different for set-user-ID programs). For `faccessat2()`, the `AT_EACCESS` flag can be used to check the effective IDs instead. As Salaün noted, though, Torvalds's suggestion was similar to what Salaün had earlier done with `AT_INTERPRETED` for `faccessat2()`; he is willing to go back to that mechanism and wondered if Torvalds liked that approach better. 

Torvalds looked at the earlier patch, which he [said](<https://lwn.net/ml/linux-kernel/CAHk-=wjPuRi5uYs9SuQ2Xn+8+RnhoKgjPEwNm42+AGKDrjTU5g@mail.gmail.com/>) was a more reasonable approach, though he had some specific questions and suggestions. He wondered why a new mode bit, perhaps called `EXECVE_OK`, could not be used instead of adding the new `AT_INTERPRETED` flag value. That way it could be used for both `access()`, which lacks a flags parameter, and for `faccessat2()`; that makes more sense given what is being checked. The currently defined mode bits for those system calls check for read, write, or execute access. 

Salaün [agreed](<https://lwn.net/ml/linux-kernel/1eeae491-7f4f-2cbc-7dbb-04e926c78b89@digikod.net/>) that using a mode bit was a better choice. Some of the other oddities that Torvalds noted in the patch were due to it being an early version of the feature on a path that was quickly abandoned after Viro's objection. Salaün plans to update the patch and resubmit, though one might guess Viro will still have the same objections, so how far it all goes is not clear. In addition, if further checks are added, such as for Linux security module (LSM) access restrictions or file-integrity verification, it may be done by way of additional bits on `fs.trusted_for_policy` (with a new name), but it will require additional code for `access()`/`faccessat2()` to actually perform the checks. 

#### Bikeshed history

Ted Ts'o [suggested](<https://lwn.net/ml/linux-kernel/YkxYHqLqTEKFrCeg@mit.edu/>) that the history of the evolution of the feature would be a good addition to the changelog: 

> As a suggestion, something that can be helpful for something which has been as heavily bike-sheded as this concept might be to write a "legislative history", or perhaps, a "bike shed history". 
> 
> And not just with links to mailing list discussions, but a short summary of why, for example, we moved from the open flag O_MAYEXEC to the faccessat(2) approach. I looked, but I couldn't find the reasoning while diving into the mail archives. [...] 
> 
> It might be that when all of this is laid out, we can either revisit prior design decisions as "that bike-shed request to support this corner case was unreasonable", or "oh, OK, this is why we need as fully general a solution as this". 

Some of that information is contained in the [patch](<https://lwn.net/ml/linux-kernel/20220104155024.48023-3-mic@digikod.net/>) that actually adds the system call, though it mostly just lists the changes for each version without a lot of explanation of the sort Ts'o is looking for. This article and the earlier two may also help fill in some of those holes. 

Overall, it is a fairly simple feature that could provide some useful functionality in specialized environments. But where it actually will live has been rather difficult to resolve. Given Torvalds's preference, returning to the plan for putting it in `access()` and `faccessat2()` looks like it has a plausible future, but we will have to see how version 19 (and beyond) of the patch set fare.
