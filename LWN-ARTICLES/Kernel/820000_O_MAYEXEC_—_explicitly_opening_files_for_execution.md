---
title: O_MAYEXEC — explicitly opening files for execution
url: https://lwn.net/Articles/820000/
date: "May 11, 2020"
category: "Filesystems-Virtual filesystem layer; Security-Language interpreters"
author: "By Jonathan Corbet May 11, 2020"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 11, 2020

Normally, when a kernel developer shows up with a proposed option that doesn't do anything, a skeptical response can be expected. But there are exceptions. Mickaël Salaün is [proposing](<https://lwn.net/ml/linux-kernel/20200505153156.925111-1-mic@digikod.net/>) the addition of a new flag (`O_MAYEXEC`) for the [`openat2()`](<http://man7.org/linux/man-pages/man2/openat2.2.html>) system call that, by default, will change nothing. But it does open a path toward tighter security in some situations. 

Executing a file on a Unix-like system requires that said file have an applicable execute-permission bit set. The file must also not reside on a filesystem that has been mounted with the `noexec` option. These checks can prevent the execution of unwanted code on a tightly controlled system, but there is a significant hole in this protection: interpreters that will happily read and execute code found in a file. If a file contains Perl code, for example, it cannot be executed by typing its name if it fails either of the above two tests. If an attacker is able to pass that file as a parameter to a `perl -e` command, though, its contents will still be executed. 

The new `O_MAYEXEC` flag is a way for language interpreters (or other programs, such as dynamic linkers, that execute code) to indicate to the kernel that a file is being opened with the intent of executing its contents. This flag is totally ignored by `open()` which, because it never checked for invalid flags, is difficult to extend in general. The newer `openat2()` system call, instead, does fail when unknown flags are passed to it; it has been extended to recognize `O_MAYEXEC`. But, by default, nothing will change if that flag is present. 

The patch set also adds a new sysctl knob called `fs.open_mayexec_enforce` that can bring out a change in behavior. Its default value (zero) naturally preserves current behavior so that nobody's system is broken by mistake. If, instead, bit 0 is set, an `openat2()` call with `O_MAYEXEC` will fail if the filesystem holding the target file was mounted with the `noexec` option. Bit 1 will cause such an open to fail if the file lacks execute permission. Setting both bits will thus cause `O_MAYEXEC` opens to fail in the situations where a direct attempt at execution would also fail. 

Integrity measurement is another subsystem that can benefit from `O_MAYEXEC`. The kernel's integrity-measurement subsystem can be configured to block the execution of files that do not meet the integrity criteria but, once again, passing a file directly to an interpreter can bypass this check. This patch set adds a hook by which files opened with `O_MAYEXEC` can be passed to the integrity-measurement code for vetting before an open is allowed to succeed. 

Finally, as one might expect, security modules can also note the existence of this flag and respond accordingly. It would be relatively straightforward to write a policy for SELinux or Smack that prevents execution-by-interpreter of files that lack a certain label (or to prevent such execution entirely, of course). 

The above discussion skips over one little detail, though: this mechanism will only work if the programs that execute code from files cooperate and provide the `O_MAYEXEC` flag. That would require getting patches into various language interpreters, linkers, etc. to properly mark the opening of any files that might lead to the execution of code. Actions such as opening files passed on the command line, importing code in modules, and more would need this flag. Getting all of the commonly installed interpreters patched is likely to be a project that takes some time, even if all of the relevant projects go along with the idea. 

The good news is that some projects, at least, are aware of the issue. The Python project, for example, has been working since (at least) 2017 to [provide audit information](<https://lwn.net/Articles/732951/>) to the underlying operating system; that work is currently formalized as [PEP 578](<https://www.python.org/dev/peps/pep-0578/>) ("Python runtime audit hooks"), which was [approved](<https://lwn.net/ml/python-dev/2f1e4bef-d845-c006-537f-d81abc4c980c%40python.org/>) in May 2019 and appears to be on track for the Python 3.9 release. Simply supporting `O_MAYEXEC` doesn't require the addition of an entire new subsystem, though, so adding this support to other interpreters need not be a multi-year project. 

This patch set is in its fifth revision as of this writing. It has changed considerably as the result of review comments. The [original version](<https://lwn.net/ml/linux-kernel/20181212081712.32347-1-mic@digikod.net/>), posted at the end of 2018, predated `openat2()` and relied on the Yama security module for enforcement. Developers seem relatively happy with the current version, though, so this feature may be getting close to being ready to enter the mainline. Only then can the task of adding support to various interpreters begin.
