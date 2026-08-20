---
title: Delegating privilege with BPF tokens
url: https://lwn.net/Articles/935195/
date: "June 22, 2023"
category: "BPF-Security"
author: "By Jonathan Corbet June 22, 2023"
---

By **Jonathan Corbet**  
June 22, 2023

The quest to enable limited use of BPF features in unprivileged processes continues. In [the previous episode](<https://lwn.net/Articles/929746/>), an attempt to use authoritative Linux security module (LSM) hooks for this purpose was strongly rejected by the LSM developers. BPF developer Andrii Nakryiko has now returned with a new mechanism based on a privilege-conveying token. That approach, too, has run into some resistance, but a solution for the strongest concerns might be in sight. 

Nakryiko (and his employer) would like the ability to allow a process to carry out a limited set of BPF operations without needing to hold any special capabilities. Currently, most BPF operations require (at least) the `CAP_BPF` capability, so code that needs to use BPF functionality must be run with privilege that often goes beyond what is actually needed. The security module implemented in Nakryiko's previous attempt could have been used to allow specific operations as controlled by the security policy, but this module required authoritative hooks (security hooks that grant access that would otherwise be denied); such hooks are not allowed in the kernel. Thus, necessarily, the new approach takes a different tack. 

In early June, Nakryiko posted [a patch set](<https://lwn.net/ml/bpf/20230602150011.1657856-1-andrii@kernel.org/>) implementing the concept of a "BPF token" that can be used to convey limited, BPF-related capabilities from one process to another. A privileged supervisor process can use a new command to the [`bpf()`](<https://man7.org/linux/man-pages/man2/bpf.2.html>) system call, `BPF_TOKEN_CREATE`, to create a token, which is returned in the form of a special file descriptor. The creator specifies the operations that the token is meant to enable; these include creating maps (with control over which types of maps can be created), loading BPF type format (BTF) data, loading programs, and creating more tokens. 

There is a flag that causes the kernel to ignore any abilities requested that do not actually exist; its purpose is to ease the task of writing code that works across multiple kernel versions, some of which may not support all operations. This option can also be used to create a token that is valid for _any_ supported operation — even those that do not exist when the code is written. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Once created, a token can be passed to another process with the usual `SCM_RIGHTS` mechanism. It is also possible to "pin" a token into the [BPF filesystem](<https://lwn.net/Articles/664688/>), making it usable to any process that is able to access that filesystem. Pinning can be a way to inject a BPF token into a running container, for example. Since the BPF filesystem is namespace-aware, pinning a token into a specific container's filesystem does not make that token globally visible. 

Most `bpf()` calls use a command-specific structure in the sprawling [`bpf_attr` union](<https://elixir.bootlin.com/linux/v6.4-rc7/source/include/uapi/linux/bpf.h#L1320>). When token support is added to a specific command, that command's structure gains a new integer field where the caller can place their token. If a token is present and grants the ability to carry out the requested operation, the request will proceed regardless of whether the calling process has the needed capabilities. As is [the case with BPF generally](<https://lwn.net/Articles/933182/>), a value of zero indicates "no file descriptor" (and thus no token), so file descriptor zero cannot be used to represent a BPF token. 

The first posting of this work drew [a response](<https://lwn.net/ml/bpf/1930272b-cfbe-f366-21ca-e9e7a51347be@schaufler-ca.com/>) from security developer Casey Schaufler, who was unenthusiastic: 

> Token based privilege has a number of well understood weaknesses, none of which I see addressed here. I also have a real problem with the notion of "trusted unprivileged" where trust is established by a user space application. Ignoring the possibility of malicious code for the moment, the opportunity for accidental privilege leakage is huge. 

Later, in response to [a request](<https://lwn.net/ml/bpf/CAEf4BzZ5adUcs1qaHx34ZuXMyG6ByczyUqpFKq=+CtxPHYgEVQ@mail.gmail.com/>) from Nakryiko, Schaufler [described](<https://lwn.net/ml/bpf/24dcbfec-1527-ab14-9726-ca91d68f35d4@schaufler-ca.com/>) some of the weaknesses he was talking about; most of them involved a token leaking out of its intended container and being abused by an attacker. Nakryiko [responded](<https://lwn.net/ml/bpf/CAEf4BzYj9YY==awasOt+ufJGJj7P2g6qC6aMxX-Phos01aUXqw@mail.gmail.com/>) that this mechanism was intended to be used in high-trust environments where the attacker shouldn't exist, but Schaufler [said](<https://lwn.net/ml/bpf/a61d8739-300f-67b0-7e7a-acf8fb1a44a8@schaufler-ca.com/>) that was inadequate, and that the security mechanism had to ensure that it could not be abused in that way. 

Undeterred, Nakryiko [posted a new version](<https://lwn.net/ml/bpf/20230607235352.1723243-1-andrii@kernel.org/>) of the patch set a few days later with only minimal changes. This time, it was Toke Høiland-Jørgensen who [raised concerns](<https://lwn.net/ml/bpf/871qik28bs.fsf@toke.dk/>) about this approach: 

> I am not convinced that this token-based approach is a good way to solve this: having the delegation mechanism be one where you can basically only grant a perpetual delegation with no way to retract it, no way to check what exactly it's being used for, and that is transitive (can be passed on to others with no restrictions) seems like a recipe for disaster. 

He went on to suggest the creation of a privileged process that could receive BPF requests via remote procedure calls and apply whatever policy made sense before executing them. Nakryiko [responded](<https://lwn.net/ml/bpf/CAEf4BzYin==+WF27QBXoj23tHcr5BeezbPj2u9RW6qz4sLJsKw@mail.gmail.com/>) that this design would not work well in practice — an answer that was [echoed](<https://lwn.net/ml/bpf/CA+khW7h95RpurRL8qmKdSJQEXNYuqSWnP16o-uRZ9G0KqCfM4Q@mail.gmail.com/>) by Hao Luo, who described Google's experience with that pattern. 

Djalal Harouni also [expressed](<https://lwn.net/ml/bpf/CAEiveUdU7On9c27iek2rRmqSLFTKduNUtjEAD0iaCPQ4wZoH6Q@mail.gmail.com/>) concerns that tokens could leak between containers, and suggested that a BPF token should be an attribute of a specific BPF filesystem instance. That, he said, would help to attach the token to a specific namespace, preventing leakage and matching how other credentials are handled. Christian Brauner [agreed](<https://lwn.net/ml/bpf/20230614-geruch-verzug-db3903a52383@brauner/>) with that suggestion. 

In response, Nakryiko [acknowledged](<https://lwn.net/ml/bpf/CAEf4BzawogpzENKC=KYk+mvc375ZF8Rs0gnu5grOywUsM0AV+Q@mail.gmail.com/>) the concern and the suggested solution: 

> The main worry is that BPF token, once issued, could be illegally/uncontrollably passed outside of container, intentionally or not. And by having this association with mount namespace (through BPF FS) we automatically limit the sharing to only contain that has access to that BPF FS. 

He suggested a slightly different implementation, though, based on his desire to allow a namespace to have more than one token: the creation of a BPF token could include a file descriptor identifying a BPF filesystem instance. The resulting token could only be pinned into that specific filesystem instance, and would be prevented, somehow, from leaving the mount namespace where that filesystem instance exists; ""specific details to be worked out"". 

[Version 3 of the patch set](<https://lwn.net/ml/bpf/20230621233809.1941811-1-andrii@kernel.org/>), posted on June 22, implemented a step in this direction. In this version, creating a token and pinning it into a BPF filesystem are done in a single operation, and it is no longer possible to pin a token after creation. That will keep tokens from being pinned outside of the intended context, but does not address the possibility that a token could be deliberately leaked via `SCM_RIGHTS`. So Nakryiko's objective that a BPF token ""cannot leave the boundaries of that mount namespace"" has not yet been fully achieved. 

Whether that change is enough to address the concerns that have been expressed remains to be seen. Then we will have to see whether the more security-oriented developers in the community are willing to accept a token-based mechanism in general. If not, it would probably be a good time for them to suggest a workable alternative. Should no such problems arise, though, BPF tokens may make an appearance in a near-future kernel release.
