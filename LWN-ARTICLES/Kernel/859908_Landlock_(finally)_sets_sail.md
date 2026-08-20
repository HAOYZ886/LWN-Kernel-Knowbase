---
title: Landlock (finally) sets sail
url: https://lwn.net/Articles/859908/
date: "June 17, 2021"
category: "Releases-5.13; Security-Security modules"
author: "By Jonathan Corbet June 17, 2021"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 17, 2021

Kernel development is not for people who lack persistence; changes can take a number of revisions and a lot of time to make it into a mainline release. Even so, the story of the [Landlock](<https://landlock.io/>) security module, developed by Mickaël Salaün, seems like an extreme case; this code was [merged for 5.13](<https://git.kernel.org/linus/17ae69aba89d>) after more than five years of development and 34 versions of the patch set. This sandboxing mechanism has evolved considerably since LWN [covered version 3 of the patch set](<https://lwn.net/Articles/703876/>) in 2016, so a look at what Landlock has become is warranted. 

Like [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>), Landlock is an unprivileged sandboxing mechanism; it allows a process to confine itself. The long-term vision has always included adding controls for a wide range of possible actions, but those in the actual patches have been limited to filesystem access. In the early days, Landlock worked by allowing a process to attach BPF programs to various security hooks in the kernel; those programs would then make access-control decisions when asked. BPF maps would be used to associate specific programs with portions of the filesystem, and a special `seccomp()` mode was used to control the whole thing. 

The goals behind Landlock have never been particularly controversial, but the implementation is a different story. The use of BPF was questioned even before making BPF available to unprivileged users in any context [fell out of favor](<https://lwn.net/Articles/796328/>). It was also felt that `seccomp()`, which controls access to system calls, was a poor fit for Landlock, which does not work at the system-call level. For some time, Salaün was encouraged by reviewers to add a set of dedicated system calls instead; it took him a while to give that approach a try. 

In the end, though, dedicated system calls turned out to be the winning formula. [Version 14](<https://lwn.net/ml/linux-kernel/20200224160215.4136-1-mic@digikod.net/>) of the patch set, posted in February 2020, dropped BPF in favor of a mechanism for defining access-control rules and added a multiplexing `landlock()` system call to put those rules into force. The [20th version](<https://lwn.net/ml/linux-kernel/20200802215903.91936-1-mic@digikod.net/>) split the multiplexer into four separate system calls, but one of those was dropped in [the next revision](<https://lwn.net/ml/linux-kernel/20201008153103.1155388-1-mic@digikod.net/>). So Landlock, as it will appear in 5.13, will bring three system calls with it. 

#### The 5.13 Landlock API

The first of those system calls creates a rule set that will be used for access-control decisions. Each rule set must be given a set of access types that it will handle. To define a rule set that can handle _all_ action types, one would start like this: 

```
struct landlock_ruleset_attr ruleset_attr = {
            .handled_access_fs =
                LANDLOCK_ACCESS_FS_EXECUTE |
                LANDLOCK_ACCESS_FS_WRITE_FILE |
                LANDLOCK_ACCESS_FS_READ_FILE |
                LANDLOCK_ACCESS_FS_READ_DIR |
                LANDLOCK_ACCESS_FS_REMOVE_DIR |
                LANDLOCK_ACCESS_FS_REMOVE_FILE |
                LANDLOCK_ACCESS_FS_MAKE_CHAR |
                LANDLOCK_ACCESS_FS_MAKE_DIR |
                LANDLOCK_ACCESS_FS_MAKE_REG |
                LANDLOCK_ACCESS_FS_MAKE_SOCK |
                LANDLOCK_ACCESS_FS_MAKE_FIFO |
                LANDLOCK_ACCESS_FS_MAKE_BLOCK |
                LANDLOCK_ACCESS_FS_MAKE_SYM,
        };
```

(This example and those that follow were all taken from [the Landlock documentation](<https://landlock.io/linux-doc/landlock-v34/userspace-api/landlock.html>)). 

Once that structure is defined, it can be used to create the rule set itself: 

```
int landlock_create_ruleset(struct landlock_ruleset_attr *attr,
        				size_t attr_size, unsigned int flags);
```

The `attr_size` parameter must be the size of the `landlock_ruleset_attr` structure (which allows for future expansion in a compatible manner); `flags` must be zero (with one exception, described below). If all goes well, the return value will be a file descriptor representing the newly created rule set. 

That set does not actually contain any rules, yet, so it is of limited utility. The 5.13 version of Landlock only supports a single type of rule, controlling access to everything contained within (and below) a given directory. The first step is to define a structure describing what accesses will be allowed for a given subtree; to limit access to reading and executing, one could use something like this: 

```
struct landlock_path_beneath_attr path_beneath = {
            .allowed_access =
                LANDLOCK_ACCESS_FS_EXECUTE |
                LANDLOCK_ACCESS_FS_READ_FILE |
                LANDLOCK_ACCESS_FS_READ_DIR,
        };
```

The `landlock_path_beneath_attr` structure also contains a field called `parent_fd` that should be set to a file descriptor for the directory where the rule is to be applied. So, for example, to limit access to `/usr` to the above operations, a process could open `/usr` as an `O_PATH` file descriptor, assigning the result to `path_beneath.parent_fd`. Finally, this rule should be added to the rule set with: 

```
int landlock_add_rule(int ruleset_fd, enum landlock_rule_type rule_type,
    			  void *rule_attr, unsigned int flags);
```

Where `ruleset_fd` is the file descriptor representing the rule set, `rule_type` is `LANDLOCK_RULE_PATH_BENEATH` (the only supported value, currently), `rule_attr` is a pointer to the structure created above, and `flags` is zero. The return value will be zero if all goes well. Multiple rules can be added to a single rule set. 

The rule set has now been defined, but is not yet active. To bind itself to a given set, a process will call: 

```
int landlock_restrict_self(int ruleset_fd, unsigned int flags);
```

Once again, `flags` must be zero. This operation will fail unless the process has previously called [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) with the `PR_SET_NO_NEW_PRIVS` operation to prevent the acquisition of capabilities through setuid programs. Multiple calls may be made to `landlock_restrict_self()`, each of which will increase the number of restrictions in force. Once a rule set has been made active, it cannot be removed for the life of the process. Rules enforced by Landlock will be applied to any child processes or threads as well. 

For the curious, there is a sample sandboxing program using Landlock that was added in [this commit](<https://git.kernel.org/linus/ba84b0bf5a16>). 

#### After 5.13

Landlock is useful in its current form, but it can be expected to gain a number of new features in future kernel releases now that the core infrastructure is in place. That could present a problem for sandboxing programs, which would like to use those newer features but must be prepared to cope with older kernels that lack them. To help future application developers, Salaün added a mechanism to help determine which features are available. If `landlock_create_ruleset()` is called with `flags` set to `LANDLOCK_CREATE_RULESET_VERSION`, it will return an integer value indicating which version of the Landlock API is supported; currently that value will always be one. When new features are added, the version number will be increased; developers will thus be able to use the version to know which features are supported on any given system. 

Landlock has clearly reached an important milestone after more than five years of work, but it seems just as clear that this story is not yet done. After perhaps taking a well-deserved break, Salaün can be expected to start fleshing out the set of Landlock features; with luck, these will not take as long to find acceptance in the kernel community. There may come a time when Landlock can do much of what `seccomp()` can do, but perhaps in a way that is easier for application developers to use.
