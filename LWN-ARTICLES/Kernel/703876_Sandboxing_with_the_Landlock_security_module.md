---
title: Sandboxing with the Landlock security module
url: https://lwn.net/Articles/703876/
date: "October 19, 2016"
category: "Security-Security modules"
author: "By Jonathan Corbet October 19, 2016 Kernel
Recipes"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 19, 2016

* * *

[Kernel Recipes](<https://lwn.net/Archives/ConferenceByYear/#2016-Kernel_Recipes>)

Anybody working to harden a computing system is likely to look at sandboxing fairly early in the process. The prospect of vulnerabilities in running software is a bit less worrisome if the scope for exploitation of those vulnerabilities is limited, and a sandbox can limit an attacker's freedom nicely. The kernel has a number of mechanisms that can support sandboxing now, and others are under development. One of those, the [Landlock security module](<https://lwn.net/Articles/700607/>), was the topic of [Mickaël Salaün's talk](<https://kernel-recipes.org/en/2016/talks/seccomp-lsm-unprivileged-sandboxing/>) at [Kernel Recipes 2016](<https://kernel-recipes.org/en/2016/>). 

The goal for Landlock, Salaün said, is to allow unprivileged users to restrict processes that they run. He is trying to create something that is similar to the OpenBSD `pledge()` ([formerly `tame()`](<https://lwn.net/Articles/651700/>)) system call. By restricting what a running process can do, a Landlock-based sandbox can reduce the attack surface of the kernel and, with luck, make the exploitation of vulnerabilities harder in general. 

Why not use the mechanisms that the kernel already provides? The Linux security module (LSM) subsystem offers mechanisms like SELinux or Smack, but those are meant for administrators, not users, Salaün said. Their configuration is complex, and setting policies is a privileged operation, which runs counter to the goal of working for unprivileged users. The `seccomp()` mechanism can be used to create sandboxes, but it is limited; only 64 bits of information can be passed to a `seccomp()` hook, and it is not possible to filter system calls [![\[Mickaël Salaün\]](https://static.lwn.net/images/conf/2016/recipes/MickaëlSalaün-sm.jpg)](<https://lwn.net/Articles/703880/>) based on the paths of files they try to access. The system-call level is also the wrong place for this kind of filtering; the security hooks used by the LSM subsystem are better placed for making proper access-control decisions. 

Thus, a new LSM. It can be thought of as being similar to `seccomp()`, in that it allows the loading of BPF programs to make access-control decisions. There are two aspects to that functionality that are of interest. 

The first is the ability to attach BPF programs directly to the LSM hook functions and to give them access to the arguments passed to the hooks. In the current form of the patch set, the `security_file_open()`, `security_file_permission()`, and `security_mmap_file()` hooks can have programs attached to them; there are plans to add more hooks in the future. 

These hooks need the ability to make access-control decisions; in particular, Salaün is looking for the ability to make path-based decisions. So, for example, a program might be blocked from accessing any files outside of a dedicated, application-specific directory. To support this type of decision-making, a new type of BPF map (`BPF_MAP_TYPE_LANDLOCK_ARRAY`) is added. These maps can hold kernel pointers with an associated type; the actual use is to hold pointers to `file` structures. Then, there is a set of new BPF-callable utility functions with convenient names like `bpf_landlock_cmp_fs_beneath_with_struct_file()` that can tell a BPF program whether one `file` structure is beneath another in the filesystem hierarchy. 

With that supporting structure in place, one can see how a Landlock-based sandbox would work. The control program populates a special map with the file descriptors (converted to `file` structures internally) of the directories that the sandboxed program is to be allowed to access. A simple BPF program, which is attached to the security hooks that are called when files are opened, can then ensure that any file-access attempt is located in or below one of the directories stored in the map. Confining a process to specific parts of the filesystem thus becomes relatively easy. 

The last remaining piece is causing the relevant BPF programs to be run for the process(es) in the sandbox. There are two different ways in which that can be done: 

  * There is a new `seccomp()` operation, called `SECCOMP_SET_LANDLOCK_HOOK`, which will cause a program to be attached to a specific LSM hook for the current process. It is possible to request that the program be invoked every time the equivalent LSM hook is called, but there is another possibility as well. A normal `seccomp()` program can be attached to one or more system calls as usual, and Landlock can be told to only run the LSM-attached program if the `seccomp()` program returns the special `SECCOMP_RET_LANDLOCK` value. The `seccomp()` program can, thus, make the access-control decision by itself, or it can decide to defer to the Landlock program(s) that will be invoked later. 

  * Landlock programs can be attached to a control group, using an extension to the [`bpf_prog_attach()`](<https://lwn.net/Articles/698073/>) patch. In this case, every process running within that control group will be regulated by the Landlock programs. 

It is worth noting that the Landlock BPF programs are stackable in either context; if multiple layers of programs are attached, each will run in order and each will have the ability to veto any given operation. 

Salaün demonstrated [a simple program](<https://lwn.net/Articles/703889/>) that uses the Landlock hooks. One need simply set the environment variable `LANDLOCK_ALLOWED` to a list of directories that a program should be allowed to access, then use the example program to launch the program of interest. The sandboxed program will be unable to access anything outside of the given list. Attempts to access forbidden files are turned back with an `EPERM` error; unlike `seccomp()`, Landlock does not kill programs that run into access restrictions. 

The response to the module thus far has been mostly positive. Andy Lutomirski is [concerned](<https://lwn.net/Articles/703879/>) about the control-group mode, though, given that there are still outstanding questions about how the version-2 control-group interface is going to work in general. So he recommends leaving that piece out and just using `seccomp()` until that issue has been resolved. The control-group hook is a tiny piece of the whole, so, if leaving it out is the price of admission for now, it is hard to imagine that anybody will be too upset. 

One other potential problem is that there is a competing proposal out there in the form of the [Checmate module](<https://lwn.net/Articles/698843/>). From all appearances, though, Landlock is further along and more actively developed. It may make sense to take ideas from both projects, though; Checmate is more focused on networking operations at the moment, which is an area that Landlock has yet to address. So the details are yet to be determined, but it seems likely that there will be some sort of BPF-based security module in the kernel before too long. It has taken a while for the stackable security modules concept to bring about a new set of interesting security mechanisms, but that would appear to be happening at last. 

[Your editor would like to thank Kernel Recipes for supporting his travel to the event.]
