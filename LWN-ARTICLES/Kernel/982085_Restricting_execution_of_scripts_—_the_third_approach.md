---
title: Restricting execution of scripts — the third approach
url: https://lwn.net/Articles/982085/
date: "July 19, 2024"
category: "Security-Language interpreters"
author: "By Jonathan Corbet July 19, 2024"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 19, 2024

The kernel will not consent to execute just any file that happens to be sitting in a filesystem; there are formalities, such as the checking of execute permission and consulting security policies, to get through first. On some systems, security policies have been established to limit execution to specifically approved programs. But there are files that are not executed directly by the kernel; these include scripts fed to language interpreters like Python, Perl, or a shell. An attacker who is able to get an interpreter to execute a file may be able to bypass a system's security policies. Mickaël Salaün has been working on closing this hole for years; the [latest attempt](<https://lwn.net/ml/all/20240704190137.696169-1-mic@digikod.net>) takes the form of a new flag to the [`execveat()`](<https://man7.org/linux/man-pages/man2/execveat.2.html>) system call. 

Imagine a locked-down system containing a file with some hostile Python code. There are many ways to try to execute that file, two of which are: 

```
$ ./bad-stuff.py
        $ python bad-stuff.py
```

In the first case, the kernel reads the [shebang line](<https://en.wikipedia.org/wiki/Shebang_\(Unix\)>) at the beginning of the file, sees that it is intended to be fed to the Python interpreter, and executes that interpreter with the file as its standard input. Before doing so, though, the kernel will ensure that the file has the execute-permission bit set and that any security policy in force does not object — just like it would with any executable file. In the latter case, though, the kernel has no way to know that `bad-stuff.py` is being opened for execution, so none of those checks are made. 

Salaün's objective is to cause both of those commands to have the same result, be it successfully executing the file or refusing to do so. His first attempt, initially [posted](<https://lwn.net/ml/linux-kernel/20181212081712.32347-1-mic@digikod.net/>) in 2018 and [extensively revised](<https://lwn.net/Articles/820000/>) in 2020, added a new flag (`O_MAYEXEC`) for `[openat()](<https://man7.org/linux/man-pages/man2/openat2.2.html>)` that an interpreter could use to indicate that a file was being opened for execute access. The patch series was not entirely well received; among other things, Salaün was [told](<https://lwn.net/ml/linux-kernel/20200909171316.GW1236603@ZenIV.linux.org.uk/>) that this functionality really should be implemented as a new system call instead. So he [tried that approach](<https://lwn.net/Articles/832959/>) next, adding a system call named `trusted_for()`, which seemed to be making progress until Linus Torvalds [refused](<https://lwn.net/ml/all/CAHk-=wgoC76v-4s0xVr1Xvnx-8xZ8M+LWgyq5qGLA5UBimEXtQ@mail.gmail.com/>) to pull it during the 5.18 merge window in 2022; he suggested, instead, adding a new flag to an existing call. 

Rather than go back to `O_MAYEXEC`, though, Salaün has taken the approach of adding a flag, `AT_CHECK`, to `execveat()`. If that flag is set, `execveat()` will perform all of the checks it normally would before executing a file, but will not actually execute it; instead, it returns a status code indicating whether the checks passed or not. Language interpreters can use that result to decide whether to execute a given script file or not. This implementation has the advantage of always performing the same checks that would apply to files executed directly by the kernel, ensuring a more consistent policy. 

Of course, whether to execute a file passed to an interpreter is a policy decision, and not all systems are the same; if Python abruptly started refusing to execute arbitrary script files, many users would understandably become somewhat irate. One solution that has been employed at times is to build a special version of every interpreter to load onto restricted systems, but that makes distribution architects and packagers irate instead. As a way of keeping the overall level of irateness down, Salaün has added a new set of securebits flags (securebits are documented toward the end of the sprawling [capabilities man page](<https://man7.org/linux/man-pages/man7/capabilities.7.html>)): 

  * `SECBIT_SHOULD_EXEC_RESTRICT`: interpreters should only consent to execute files that are approved by system policy. That approval may take the form of an explicit allowlist, cryptographic signatures, or something else. 
  * `SECBIT_SHOULD_EXEC_CHECK`: interpreters should use the new `execveat()` flag to check whether a given file should be executed. 

There are also `_LOCKED` versions of these bits that prevent them from being changed in the future. All of the bits are inherited by child processes, so if they are (for example) set in a container's init process, they will be set in every other process running in the container as well. 

The actual semantics of these bits caused a bit of confusion in the ensuing discussion. In a normal locked-down mode, both bits would be set to enable the full set of checks. Setting only `SECBIT_SHOULD_EXEC_CHECK` can be useful for logging and tracking down problems, similar to the SELinux permissive mode. Setting only `SECBIT_SHOULD_EXEC_RESTRICT`, instead, leads to a situation where almost nothing can be executed by the interpreter (though files with shebang lines would still work if the system otherwise allows them). Leaving both bits clear results in a system that works like normal, open systems do now. 

After some discussion, Salaün responded to the confusion by [proposing](<https://lwn.net/ml/all/20240710.eiKohpa4Phai@digikod.net/>) a different mode of operation. Rather than a pair of bits telling interpreters that they should be restricting access, simply instruct interpreter developers to always perform the `AT_CHECK` check; a different pair of securebits would instruct the interpreter about which policy to apply: 

  * `SECBIT_EXEC_RESTRICT_FILE`: files can only be executed by the interpreter if the `AT_CHECK` call succeeds. While the kernel _could_ interpret this bit, the proposal is still to defer this enforcement to user space, and to have the `AT_CHECK` call return the same results regardless of the securebits settings. 
  * `SECBIT_EXEC_DENY_INTERACTIVE`: the interpreter should refuse to run any "interactive" input that comes from its command line, environment variables, configuration files, etc. Only code that exists in an executable file on disk could be run. 

Once again, leaving both bits clear results in an open system. Setting only `SECBIT_EXEC_RESTRICT_FILE` would prevent the accidental execution of files, but still puts trust in the users. Setting only `SECBIT_EXEC_DENY_INTERACTIVE` would allow any file to be executed, but could be used as a sort of permissive mode on the way toward locking down the system entirely, which would be accomplished by setting both bits. 

Early in the discussion, Kees Cook had [suggested](<https://lwn.net/ml/all/202407041711.B7CD16B2@keescook/>) that the original `O_MAYEXEC` approach may have been better; since it refuses to open the file if the check failed, there was no way to trick an interpreter into executing the file anyway. Later on, though, he [indicated](<https://lwn.net/ml/all/202407100921.687BE1A6@keescook/>) support for the redefined set of securebits. The discussion is ongoing, but there does not appear to be strong opposition to that approach. 

So it looks like the way might finally be clear for this functionality to enter the kernel, perhaps as soon as the 6.12 development cycle. But, then, `trusted_for()` appeared to have reached a consensus until it ended up blocked at the final stage. So it is too soon to be holding one's breath, but the third time may yet prove to be the charm for this feature.
