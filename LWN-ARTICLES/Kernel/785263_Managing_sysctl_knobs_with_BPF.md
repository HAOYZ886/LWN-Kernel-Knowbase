---
title: Managing sysctl knobs with BPF
url: https://lwn.net/Articles/785263/
date: "April 9, 2019"
category: BPF; Sysctl
author: "By Jonathan Corbet April 9, 2019"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 9, 2019

"Sysctl" is the kernel's mechanism for exposing tunable parameters to user space. Every sysctl knob is presented as a virtual file in a hierarchy under `/proc/sys`; current values can be queried by reading those files, and a suitably privileged user can change a value by writing to its associated file. What happens, though, when a system administrator would like to limit access to sysctl, even for privileged users? Currently there is no solution to this problem other than blocking access to `/proc` entirely. That may change, though, if [this patch set](<https://lwn.net/ml/linux-kernel/cover.1554485409.git.rdna@fb.com/>) from Andrey Ignatov makes its way into the mainline. 

The use case that Ignatov has in mind is containerized applications that, for one reason or another, are run as root. If `/proc` is mounted in the namespace of such a container, it can be used to change sysctl knobs for the entire system. A hostile container could take advantage of that ability for any of a number of disruptive ends, including perhaps breaking the security of the system as a whole. While disabling or unmounting `/proc` would close this hole, it may have other, unwanted effects. This situation leads naturally to the desire to exert finer-grained control over access to `/proc/sys`. 

In recent years, one would expect such control to be provided in the form of a new hook for a BPF program, and one would not be disappointed in this case. The patch set adds a new BPF program type (`BPF_PROG_TYPE_CGROUP_SYSCTL`) and a new operation in the [`bpf()` system call](<http://man7.org/linux/man-pages/man2/bpf.2.html>) (`BPF_CGROUP_SYSCTL`) to install programs of that type. As can be inferred from the names, these programs are attached by way of control groups, so different levels of control can be applied in different parts of the system. 

Once attached, the program will be invoked whenever a process in the affected control group attempts to read or write a sysctl knob. The context passed to these programs contains a flag indicating whether a write operation is being performed and the position within the sysctl file that is being read or written. To learn more, the program must call a set of new helper functions, starting with: 

```
int bpf_sysctl_get_name(struct bpf_sysctl *ctx, char *buf, size_t buf_len, 
    			    u64 flags);
```

to get the name of the knob that is being changed. By default, the full name of the knob from the root of the sysctl hierarchy (i.e. without "`/proc/sys`") is returned; the `BPF_F_SYSCTL_BASE_NAME` flag can be used to get only the last component of the name. If the program returns a value of one, the access will be allowed; otherwise it will fail with an `EPERM` error. 

That is enough for any program that just needs to filter based on the name of the knob being accessed. For more nuanced control, there is another set of helpers: 

```
int bpf_sysctl_get_current_value(struct bpf_sysctl *ctx, char *buf, size_t buf_len);
        int bpf_sysctl_get_new_value(struct bpf_sysctl *ctx, char *buf, size_t buf_len);
        int bpf_sysctl_set_new_value(struct bpf_sysctl *ctx, const char *buf, size_t buf_len);
```

The first two functions will return the current value of the knob and, for write accesses, the new value that the process in question would like to set. The BPF program can choose to allow a sysctl knob to be changed but modify the actual value being written with `bpf_sysctl_set_new_value()`. 

That is about it for the new API; sysctl is a simple subsystem, so imposing a controlling layer does not involve a lot of complexity. 

As Kees Cook [noted](<https://lwn.net/ml/linux-kernel/CAGXu5jJGC1kB-0e3DimBSprcgEBnWRB=i287g-9T5a-RidFqBg@mail.gmail.com/>), though, this proposal does raise an interesting question. He pointed out that this functionality seems more appropriate for a Linux security module (LSM) than a BPF program; LSMs exist to perform just this sort of fine-grained access control. Alexei Starovoitov [replied](<https://lwn.net/ml/linux-kernel/20190406170257.qlptcrfth2rb3rxo@ast-mbp.dhcp.thefacebook.com/>) that there is an important difference: the BPF program is tied to a specific control group, while LSMs are global across the system. That difference is important: it's what allows the administrator to set different policies for different control groups. 

That, in turn, points out a significant limitation for LSMs in general: they were designed years before control groups were added to the system, so the two features do not always play well together. LSMs can do a lot to prevent containers from running amok across the system, but they are not equipped to easily enforce different policies for different containers. A hook for a BPF program is rather more flexible in that regard. The ability to change the value written to a sysctl knob is also something that one would not find in an LSM, the job of which is to make a simple decision on whether to allow an operation to proceed or not. 

And that, perhaps, highlights part of why BPF has been so successful in recent years. The kernel's role is to enforce policy, but to allow the system administrator to say what that policy should be. In an attempt to provide sufficient flexibility, the kernel has grown elaborate frameworks for the expression of policy, including the LSM subsystem or, for example, the netfilter mechanism. But users always come up with ideas for policies that are awkward (or impossible) to express with those frameworks; they're users, that's their job. So, over time, these in-kernel policy machines grow bigger, more complicated and, often slower — and still don't do everything users would like. 

It is far easier for the kernel to provide a hook for a BPF program in places where policy decisions need to be made; a BPF hook can replace a lot of kernel code. The result also tends to be much more flexible, and it will almost certainly perform better. So it's not surprising that the kernel seems to be growing BPF hooks in all directions. The sysctl hook is just another example of how the kernel's API is being transformed by BPF; expect a lot more of these hooks to be added in the future.
