---
title: KRSI — the other BPF security module
url: https://lwn.net/Articles/808048/
date: "December 27, 2019"
category: "BPF-Security; Security-Security modules"
author: "By Jonathan Corbet December 27, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 27, 2019

One of the first uses of the [BPF virtual machine](<https://lwn.net/Articles/740157/>) outside of networking was to implement access-control policies for the [`seccomp()`](<http://man7.org/linux/man-pages/man2/seccomp.2.html>) system call. Since then, though, the role of BPF in the security area has not changed much in the mainline kernel, even though BPF has evolved considerably from the "classic" variant still used with `seccomp()` to the "extended" BPF now supported by the kernel. That has not been for a lack of trying, though. The out-of-tree Landlock security module was [covered here](<https://lwn.net/Articles/703876/>) over three years ago. We also [looked at](<https://lwn.net/Articles/798157/>) the kernel runtime security instrumentation (KRSI) patch set in September. KP Singh has posted [a new KRSI series](<https://lwn.net/ml/linux-kernel/20191220154208.15895-1-kpsingh@chromium.org/>), so the time seems right for a closer look. 

While KRSI is implemented as a Linux security module and is able to make access-control decisions, access control does not appear to be the core goal behind this work. Instead, KRSI exists to keep an eye on system behavior overall in order to detect attacks. It is, in a sense, better thought of as an extension of the kernel's audit mechanism that uses BPF to provide a higher level of configurability beyond what the audit subsystem can do. 

The concept behind KRSI is simple enough: it allows a suitably privileged user to attach a BPF program to any of the hundreds of hooks provided by the Linux security module subsystem. To make this attachment easy, KRSI exports a new filesystem hierarchy under `/sys/kernel/security/bpf`, with one file for each hook. The `[bpf()](<http://man7.org/linux/man-pages/man2/bpf.2.html>)` system call can be used to attach a BPF program (of the new type `BPF_PROG_TYPE_LSM`) to any of these hooks; there can be more than one program attached to any given hook. Whenever a security hook is called, all attached BPF programs will be called in turn; if any BPF program returns an error status, then the requested action will be denied. 

Many readers will be thinking that this mechanism sounds a lot like Landlock. While the fundamental idea — attaching BPF programs to security-module hooks — is the same, the underlying goals are different, and that leads to a different implementation. KRSI is a tool for system administrators who are interested in monitoring the behavior of the system as a whole; attaching a BPF program requires the `CAP_SYS_ADMIN` capability. Landlock, instead, is intended to allow unprivileged users to sandbox programs that they are running, so no privilege is needed to attach a BPF program to a hook via Landlock. 

This difference fundamentally affects how these modules execute. Consider, for example, the hook that the kernel calls in response to an `mprotect()` call from user space: 

```
int security_file_mprotect(struct vm_area_struct *vma, unsigned long reqprot,
    			       unsigned long prot);
```

In KRSI, the three parameters to this hook will be passed directly to any attached BPF programs; those programs can follow the `vma` pointer to learn all about the affected memory area. They can also follow `vma->vm_mm` to get to the calling processes top-level memory-management data (the `mm_struct` structure). There is, in short, a lot of information available to these programs. 

The Landlock situation is different. Since Landlock hooks are under the control of unprivileged users, they cannot be allowed to just wander through kernel data structures. So a Landlock hook for `mprotect()` is passed a structure like this: 

```
struct landlock_ctx_mem_prot {
            __u64 address;
    	__u64 length;
    	__u8 protections_current;
    	__u8 protections_requested;
        };
```

In other words, the information passed to this hook contains nothing that user space did not already know. That makes it safe for the intended use case, but is likely to be too limiting for the global auditing case. 

The advent of speculative-execution vulnerabilities, along with other factors, has led to a slow simmer of questions about whether it can ever be safe to allow unprivileged users to run extended BPF code in the kernel. The BPF developers themselves have been [coming to the conclusion](<https://lwn.net/Articles/796328/>) that it cannot be done, and have scaled back their plans to make unprivileged BPF available. Indeed, even Mickaël Salaün, the author of Landlock, now [feels](<https://lwn.net/ml/linux-kernel/a6b61f33-82dc-0c1c-7a6c-1926343ef63e@digikod.net/>) that ""it is quite challenging to safely expose eBPF to malicious processes"". He went on to say: 

I'm working on a version of Landlock without eBPF, but still with the initial sought properties: safe unprivileged composability, modularity, and dynamic update. I'll send this version soon. 

So, while it may tempting to see KRSI and Landlock as being in competition with each other, that does not really appear to be the case. 

There does not appear to be any fundamental opposition to KRSI — so far — but Casey Schaufler did [raise](<https://lwn.net/ml/linux-kernel/95036040-6b1c-116c-bd6b-684f00174b4f@schaufler-ca.com/>) the inevitable concern with this approach: ""This effectively exposes the LSM hooks as external APIs. It would mean that we can't change or delete them."" API issues often come up around BPF, especially in the tracing area, so it is unsurprising that this question would arise here. In this case, Singh [replied](<https://lwn.net/ml/linux-kernel/CACYkzJ5nYh7eGuru4vQ=2ZWumGPszBRbgqxmhd4WQRXktAUKkQ@mail.gmail.com/>): ""we *do not* want to make LSM hooks a stable API and expect the eBPF programs to adapt when such changes occur"". It has repeatedly been made clear, though, that such expectations do not override the kernel's stable-ABI rules. Given the power that would be available to KRSI hooks, it is reasonable to expect that they would be used for a range of purposes far beyond those envisioned by its developers. If unrelated kernel changes break the resulting programs, there is a good chance that they would be reverted. 

Additionally, one could argue that this kind of problem is more likely to come about with KRSI than with, for example, tracepoints. While tracepoints have been added as an explicit way to make specific information available to user space, the security-module hooks were designed for internal use. They expose a lot of information, in internal-kernel formats, that one might otherwise not choose to make available even to privileged users. That can only make them more likely to break as those kernel data structures change. Changes to the security hooks are not that common, but they do happen ([example](<https://git.kernel.org/linus/e3f20ae21079>)); developers are unlikely to react well to the idea that they would no longer be able to make that kind of change. 

The ABI issue could thus end up being the biggest obstacle to the merging of KRSI, even though such concerns have not (yet) stalled efforts in areas like tracing. It will be interesting to see what happens as the awareness of this functionality spreads. The usefulness of KRSI seems clear, but the potential for long-term problems it could bring is rather murkier.
