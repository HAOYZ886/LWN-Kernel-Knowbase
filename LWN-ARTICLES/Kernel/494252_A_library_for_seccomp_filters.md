---
title: A library for seccomp filters
url: https://lwn.net/Articles/494252/
date: "April 25, 2012"
category: "Security-seccomp"
author: "By Jake Edge April 25, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
April 25, 2012

Now that it is looking like Linux will be getting an enhanced "secure computing" (seccomp) facility, some are starting to turn toward actually using the new feature in applications. To that end, Paul Moore has [introduced libseccomp](<https://lwn.net/Articles/491308/>), which is meant to make it easier for applications to take advantage of the packet-filter-based seccomp mode. That will lead to more secure applications that can permanently reduce their ability to make "unsafe" system calls, which can only be a good thing for Linux application security overall. 

Enhanced seccomp has taken a somewhat tortuous path toward the mainline--and it's not done yet. Will Drewry's [BPF-based solution](<https://lwn.net/Articles/475043/>) (aka seccomp filter or seccomp mode 2) is currently in linux-next, and recent complaints about it have been few and far between, so it would seem likely that it will appear in the 3.5 kernel. It will provide fine-grained control over the system calls that the process (and its children) can make. 

What [libseccomp](<http://sourceforge.net/projects/libseccomp/>) does is make it easier for applications to add support for sandboxing themselves by providing a simpler API to use the new seccomp mode. By way of contrast, Kees Cook [posted a seccomp filter tutorial](<http://outflux.net/teach-seccomp/>) that describes how to build an application using the filters directly. In addition, it is also interesting to see that the recent [OpenSSH 6.0 release](<http://www.openssh.com/txt/release-6.0>) contains support for seccomp filtering using a (pre-libseccomp) [patch](<http://hg.mindrot.org/openssh/rev/f40779d28db5>) from Drewry. The patch limits the privilege-separated OpenSSH child process to a handful of legal system calls, while setting up `open()` to fail with an `EACCESS` error 

As described in the man pages that accompany the [libseccomp code](<http://sourceforge.net/p/libseccomp/libseccomp/ci/47938254136bed475ce95c155a04bcca84218b45/tree/>), the starting point is to include `seccomp.h`, then an application must call: 

```
int seccomp_init(uint32_t def_action);
```

The `def_action` parameter governs the default action that is taken when a system call is rejected by the filter. `SCMP_ACT_KILL` will kill the process, while `SCMP_ACT_TRAP` will cause a `SIGSYS` signal to be issued. There are also options to force rejected system calls to return a certain error (`SCMP_ACT_ERRNO(errno)`), to generate a `ptrace()` event (`SCMP_ACT_TRACE(msg_num)`), or to simply allow the system call to proceed (`SCMP_ACT_ALLOW`). 

Next, the application will want to add its filter rules. Those rules can apply to any invocation of a particular system call, or it can restrict calls to only use certain values for the system call arguments. So, a rule could specify that `write()` can only be used on file descriptor 1, or that `open()` is forbidden, for example. The interface for adding rules is: 

```
int seccomp_rule_add(uint32_t action,
                             int syscall, unsigned int arg_cnt, ...);
```

The `action` parameter uses the same action macros as are used in `seccomp_init()`. The `syscall` argument is the system call number of interest for this rule, which could be specified using `__NR_syscall` values, but it is recommended that the `SCMP_SYS()` macro be used to properly handle multiple architectures. The `arg_cnt` specifies the number of rules that are being passed; those rules then follow. 

In the simplest case, where the rule is just allowing a system call for example, there are no argument rules. So, if the default action is to kill the process, adding a rule to allow `close()` would look like: 

```
seccomp_rule_add(SCMP_ACT_ALLOW, SCMP_SYS(close), 0);
```

Doing filtering based on the system call arguments relies on a set of macros that specify the argument of interest by number (`SCMP_A0()` through `SCMP_A5()`), and the comparison to be done (`SCMP_CMP_EQ`, `SCMP_CMP_GT`, and so on). So, adding a rule that allows writing to `stderr` would look like: 

```
seccomp_rule_add(SCMP_ACT_ALLOW, SCMP_SYS(write), 1, 
                         SCMP_A0(SCMP_CMP_EQ, STDERR_FILENO));
```

Once all the rules have been added, the filter is loaded into the kernel (and activated) with: 

```
int seccomp_load(void);
```

The internal library state that was used to build the filter is no longer needed after the call to `seccomp_load()`, so it can be released with a call to: 

```
void seccomp_release(void);
```

There are a handful of other functions that libseccomp provides, including two ways to extract the filter code from the library: 

```
int seccomp_gen_bpf(int fd);
        int seccomp_gen_pfc(int fd);
```

Those functions will write the filter code in either kernel-readable BPF or human-readable Pseudo Filter Code (PFC) to `fd`. One can also set the priority of system calls in the filter. That priority is used as a hint by the filter generation code to put higher priority calls earlier in the filter list to reduce the overhead of checking those calls (at the expense of the others in the rules): 

```
int seccomp_syscall_priority(int syscall, uint8_t priority);
```

In addition, there are a few attributes for the seccomp filter that can be set or queried using: 

```
int seccomp_attr_set(enum scmp_filter_attr attr, uint32_t value);
        int seccomp_attr_get(enum scmp_filter_attr attr, uint32_t *value);
```

The attributes available are the default action for the filter (`SCMP_FLTATR_ACT_DEFAULT`, which is read-only), the action taken when the loaded filter does not match the architecture it is running on (`SCMP_FLTATR_ACT_BADARCH`, which defaults to `SCMP_ACT_KILL`), or whether `PR_SET_NO_NEW_PRIVS` is turned on or off before activating the filter (`SCMP_FLTATR_CTL_NNP`, which defaults to `NO_NEW_PRIVS` being turned on). The `NO_NEW_PRIVS` flag is a recent kernel addition that stops a process and its children from ever being able to get new privileges (via `setuid()` or capabilities for example). 

The last attribute came about after some discussion in the announcement thread. The consensus on the list was that it was desirable to [set `NO_NEW_PRIVS`](<https://lwn.net/Articles/475678/>) by default, but allow libseccomp users to override that if desired. Other than some kudos from other developers about the project, the only other messages in the thread concerned the GPLv2 license. Moore said that the GPL was really just his default license and, since it made more sense for a library to use the LGPL, he was able to get the other contributors to agree to switch to the LGPLv2.1 

While it is by no means a panacea, the seccomp filter will provide a way for applications to make themselves more secure. In particular, programs that handle untrusted user input, like the Chromium browser which was the original impetus for the feature, will be able to limit the ways in which damage can be done through a security hole in their code. One would guess we will see more applications using the feature via libseccomp. Seccomp mode 2 is currently available in Ubuntu kernels, and is slated for inclusion in ChromeOS--with luck we'll see it in the mainline soon too.
