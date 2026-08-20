---
title: Unprivileged BPF and authoritative security hooks
url: https://lwn.net/Articles/929746/
date: "April 27, 2023"
category: "BPF-Security; Modules-Security modules; Security-Security modules"
author: "By Jonathan Corbet April 27, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 27, 2023

When the developers of the Linux security module (LSM) subsystem find themselves disagreeing with other kernel developers, it tends to be because those other developers don't think to — or don't want to — add security hooks to their shiny new subsystems. Sometimes, though, the addition of new hooks by non-LSM developers can also create some friction. Andrii Nakryiko's posting of [a pair of BPF-related security hooks](<https://lwn.net/ml/bpf/20230412043300.360803-1-andrii@kernel.org/>) raised a couple of interesting questions, one of which spurred a fair amount of discussion, and one that did not. 

Nakryiko proposed the addition of two new LSM hooks to control access to BPF functionality. The [first](<https://lwn.net/ml/bpf/20230412043300.360803-5-andrii@kernel.org/>) would govern the creation of BPF maps, while the [second](<https://lwn.net/ml/bpf/20230412043300.360803-8-andrii@kernel.org/>) was meant to control the loading of BPF type format (BTF) data that describes functions and data structures within the kernel. The plan is to not stop there, though: 

> This patch set implements and demonstrates an overall approach starting with BPF map and BTF object creation, first two steps in the lifetime of a typical BPF applications. Next step would be to do similar changes for BPF_PROG_LOAD command to allow BPF program loading and verification. 

There is nothing in this part of the plan that is inherently controversial; if there are use cases for access control over these features beyond checking for the `CAP_BPF` capability, then the addition of these hooks to enable the creation of a policy to implement that control can make sense. But that is not quite how these hooks are meant to operate. Instead, they can be used to _bypass_ the `CAP_BPF` check entirely, meaning that they can make the covered functionality available to processes that lack that capability. 

#### Authoritative hooks

The LSM subsystem has its origin in [the first Kernel Summit](</2001/features/KernelSummit/>) in 2001. At that time, there was a desire to get an early version of SELinux into the kernel, but Linus Torvalds pointed out that there were other approaches to increased security under development, and he did not want to commit the kernel to any one of them. Instead, he asked for the creation of a framework that would allow multiple security mechanisms to be supported. 

That framework, implementing an extensive set of hooks that can make security decisions at the relevant points in the system-call paths, eventually was merged as the Linux security module subsystem. But, before that could happen, there was a heated discussion ([covered in LWN at the time](</2001/1108/kernel.php3>)) over whether the LSM subsystem should support hooks that could grant privileges that a process did not have, or whether they would only be able to add restrictions to those already implemented by the kernel's other access-control mechanisms. In the end, the decision was made that "authoritative hooks" — those that could increase privilege — would not be allowed. Among other things, this rule was seen as a way of keeping security modules from introducing security holes in their own right. 

There have been a number of security modules added in the 21 years since that decision was made, but they have all been held to that rule. Easing the ban on authoritative hooks has occasionally been discussed over those years, but has never really been considered. So, when Nakryiko proposed adding a couple of authoritative hooks, LSM maintainer Paul Moore quickly [responded](<https://lwn.net/ml/bpf/CAHC9VhQHmdZYnR=+rX-3FcRh127mhJt=jAnototfTiuSoOTptg@mail.gmail.com/>): 

> One of the hallmarks of the LSM has always been that it is non-authoritative: it cannot unilaterally grant access, it can only restrict what would have been otherwise permitted on a traditional Linux system. Put another way, a LSM should not undermine the Linux discretionary access controls, e.g. capabilities. 

The real solution, he said, would be to revise how the BPF code implements the `CAP_BPF` capability. Kees Cook [disagreed](<https://lwn.net/ml/bpf/6436eea2.170a0220.97ead.52a8@mx.google.com/>), suggesting that these hooks could be seen as ""fine-grained access control"" rather than actually bypassing enforcement, but Moore [stood firm](<https://lwn.net/ml/bpf/CAHC9VhR6ebsxtjSG8-fm7e=HU+srmziVuO6MU+pMpeSBv4vN+A@mail.gmail.com/>) in his opposition to the idea. 

Nakryiko [protested](<https://lwn.net/ml/bpf/CAEf4BzY9GPr9c2fTUS6ijHURtdNDL4xM6+JAEggEqLuz9sk4Dg@mail.gmail.com/>) that the idea was to increase security by making it finer-grained than the single `CAP_BPF` capability allows. The restriction-only model, he [said](<https://lwn.net/ml/bpf/CAEf4BzaRkAtyigmu9fybW0_+TZJJX2i93BXjiNUfazt2dFDFbQ@mail.gmail.com/>), would be more brittle in the end. He also [added](<https://lwn.net/ml/bpf/CAEf4BzZa26JHa=gBgMm-sqyNy_S71-2Rs_-F6mrRXQF9z9KcmA@mail.gmail.com/>) that there are a couple of real problems with capability-based enforcement when user namespaces are involved. The first is that many BPF programs, such as those that interact with tracing, inherently have a view of the entire system and cannot really be contained within a namespace. So a capability check for `CAP_BPF` cannot be namespace-aware. 

Beyond that, though, it is currently not even possible to give a process `CAP_BPF` if it's running within a user namespace due to the way that the capability checks are implemented in the BPF subsystem. As a result, he argued, it is not really possible for programs running within a user namespace to make use of BPF at all. The proposed hooks were intended to provide a way around this shortcoming. 

Casey Schaufler, who had been in favor of authoritative hooks back in 2001, [was unsympathetic](<https://lwn.net/ml/bpf/ad70ee53-c774-6b50-33fc-d4568a3b5559@schaufler-ca.com/>) now: 

> This doesn't sound like a problem, it sounds like BPF is explicitly designed to prevent interference by namespaces. But in some cases you now want to limit it by namespaces. 
> 
> It appears that the desired uses of BPF are no longer compatible with its original security model. That's unfortunate, and likely to require a significant change to the implementation of BPF. 

Or, as Moore [put it](<https://lwn.net/ml/bpf/CAHC9VhRH6Z2r_A7YkDEmW7kiCA8e5j2u270gE48jpQmqS+t75A@mail.gmail.com/>): ""Changing the very core behavior of the LSM layer in order to work around an issue with another access control mechanism is a non-starter"". Nakryiko has [received the message](<https://lwn.net/ml/bpf/CAEf4Bzb4TW+sYSGLFVxsvYsEHM0cXsEA2aVtaiT2QQCuQ+fnDw@mail.gmail.com/>) and has promised to come back with a different approach. It thus seems that a complete solution to the problems encountered by the BPF community is a somewhat distant prospect at this point. 

#### Unprivileged BPF

The quiet part of the discussion is an apparent change within the BPF community with regard to security. Quoting again from Nakryiko's cover letter: 

> Such LSM hook semantics gives ability to have safer-by-default policy of not giving applications any of the CAP_BPF/CAP_PERFMON/CAP_NET_ADMIN capabilities, normally required to be able to use BPF subsystem in the kernel. Instead, all the BPF processes could be left completely unprivileged, and only allowlisted exceptions for trusted and verified production use cases would be granted permission to work with bpf() syscall, as if those application had root-like capabilities. 

In the early days of extended BPF, [some effort](<https://lwn.net/Articles/660331/>) went into making it possible to use BPF without any special privileges. By 2019, though, the idea of unprivileged BPF use had been [explicitly deprecated](<https://lwn.net/Articles/796328/>). BPF co-maintainer Alexei Starovoitov described Linux as ""a single-user system"" and proclaimed that no further attempts would be made to enable use of BPF without privilege. The amount of pain involved in keeping the system secure had simply become too much; the advent of the Spectre vulnerabilities just made things worse. 

So it is interesting to see the BPF developers talking about unprivileged operation again, even if done under the watchful eye of a security policy. There does not appear to have been any discussion on the BPF list about changes in the privilege model overall, so it is not entirely clear how this all came about. 

What does seem clear is that, if the BPF developers want to move away from the simple `CAP_BPF` check, they are going to have to revisit many of the security-related decisions that they have made so far. The method of adding authoritative LSM hooks does not appear to be viable for mainline inclusion, so some thought is going to have to be put into other solutions, including perhaps rethinking the user-namespace issue. This does not look like a problem that is amenable to a quick solution.
