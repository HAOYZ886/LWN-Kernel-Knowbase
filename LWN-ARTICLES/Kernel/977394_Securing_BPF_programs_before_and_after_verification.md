---
title: Securing BPF programs before and after verification
url: https://lwn.net/Articles/977394/
date: "June 11, 2024"
category: "BPF-Security"
author: "By Daroc Alden June 11, 2024 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Daroc Alden**  
June 11, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

BPF is in a unique position in terms of security. It runs in a privileged context, within the kernel, and can have access to many sensitive details of the kernel's operation. At the same time, unlike kernel modules, BPF programs aren't signed. Additionally, the mechanisms behind BPF present challenges to implementing signing or other security features. Three nearly back-to-back sessions at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) addressed some of the potential security problems. 

#### Signing

The first session, led by KP Singh, dealt with the problem of validating signed BPF programs — although Singh preferred to call them "trusted", since the signature is only a representaton of that fact. It is difficult to verify a signed BPF program, because they are transformed in several ways prior to being loaded. A user-space loading program reads a BPF ELF file from disk and then performs relocations on it, to prepare it to run. These relocations are the mechanism behind BPF's "compile once - run everywhere" (CO-RE) support. 

[ ![\[KP Singh\]](https://static.lwn.net/images/2024/kp-singh-small.png) ](<https://lwn.net/Articles/977792>)

Unfortunately, CO-RE means that by the time the kernel sees the program, it has been altered by user space in a way that would invalidate any signatures. Even if the initial ELF file were signed, the version of the program sent to the kernel for loading would not match. Singh's answer to this problem is to use a trusted BPF loader. There are existing mechanisms to check that programs haven't been altered, notably [ fs-verity](<https://www.kernel.org/doc/html/latest/filesystems/fsverity.html>). If the kernel only accepts BPF programs from trusted user-space programs that are themselves signed, included on an fs-verity protected filesystem the kernel trusts, or verified in some other way, and _those_ programs verify BPF programs in some way before loading them, then everything should remain secure. Singh shared a demo of a simple loader that verifies BPF programs using fs-verity, to demonstrate that the idea was workable. 

There are some complications with this approach, though. For one, as the recent [ XZ backdoor](<https://lwn.net/Articles/967192/>) illustrated, dynamically loaded programs are not safe unless all of their dynamic dependencies are safe. Therefore, Singh recommended that trusted BPF loaders be statically linked. Possibly a dynamically linked program could be safe if all of the necessary libraries were also signed, but for his use case, a statically linked loader is simpler. One member of the audience objected to that, saying that systemd has started to use BPF, is interested in signing, but can't really be made to use a static binary. Singh acknowledged that there were good reasons to allow dynamic linking, but that it wasn't something he had thought about in depth. Regardless of static or dynamic linking, the important thing is that the entire path from disk to a loaded BPF program must be trusted, he said. 

Neill Kapron led a later session that challenged Singh's approach. Kapron works on Android, which has a vested interest in ensuring that the operating system can start from a trusted image. The project's current approach is to use a trusted loader, early in boot, to load any necessary BPF programs, but Kapron would like to move away from that. 

BPF is used for several different purposes in Android, he said, and networking, system, and vendor BPF programs all have their own separate update and release timelines. Currently, the complexity of running BPF programs from multiple sources across multiple kernel versions is handled using an android-specific BPF library. Kapron would like to switch to upstream libbpf instead, but can't do so until there's an answer to the security problems around loading BPF. 

Kapron considered several approaches, including a single trusted loader, signed shared library objects for libbpf, a "relocation playbook", and several others. Eventually, he settled on a different approach: moving the loading of BPF programs into the kernel. If relocations could be performed in the kernel, then the bytes read from disk could be signed using fs-verity, which would let the kernel ensure they had not been tampered with as long as the file system itself is trusted. Kapron suggested an approach where a user-space program could present a file descriptor to a file on an fs-verity filesystem and the kernel would handle the rest. 

There is a lot of support needed in the kernel for that, however. The kernel already knows how to open and read some parts of an ELF file, but the parser would need to be extended to other parts of the file format. The kernel would need to be able to create the BPF maps a program calls for, perform the relocations, and handle CO-RE. This is made more difficult because "we don't have a standard for the ELF format", Kapron explained. The existing BPF format is an ad-hoc contract between the user-space loader and the compiler. So libbpf has documented some aspects, but other libraries could do things differently. One audience member volunteered that the [ Go project](<https://go.dev/>) is changing its BPF loader to align with what libbpf does, so it might actually be a de-facto standard. 

Kapron listed some benefits of moving BPF loading into the kernel, noting that it would solve the problem of different BPF libraries having different loading behaviors, enable fully verified boots, and could even be used to permit BPF preloading — where BPF files are embedded directly into the kernel during the build. 

#### Security

Ensuring that BPF programs are not tampered with before loading was only one of the security topics discussed in the BPF track. Maxwell Bland led a session discussing other security concerns around the BPF subsystem. Bland listed verification bugs, exploit chaining, and unprivileged misuse. Verification bugs can be relevant for security because BPF depends on the verifier to ensure BPF programs' access to kernel memory is safe. Exploit chaining refers to attacks that use a program or tool to set up the next stage of an attack, rather than attacking the program or tool directly. For example, rather than targeting BPF itself, such attacks might try to use BPF to store the payload for a [ heap-spray attack](<https://en.wikipedia.org/wiki/Heap_spraying>) into kernel memory. And unprivileged misuse refers to user programs that take advantage of intended BPF features in a way that lets them exceed imposed limits. 

[ ![\[Maxwell Bland\]](https://static.lwn.net/images/2024/maxwell-bland-small.png) ](<https://lwn.net/Articles/977794>)

There is one potential problem that Bland paid particular attention to: modifying BPF programs as they are being loaded. There are only three kernel subsystems (not counting any possible modules) that violate the assumption that pages containing executable code have never been writable in the past, Bland said: BPF, kprobe self-patching, and the kernel's fixed map. These are not violations of a "write xor execute" policy, because no page in the kernel is ever simultaneously writable and executable. But if an exploit can write to a page before it is made executable, that is nearly as good. 

For BPF, this means that exploits might try to exploit other write-gadgets (parts of existing code that can be misused to write to memory) in the kernel to overwrite a page while the just-in-time (JIT) compiler is also writing to it. This isn't something that can be fixed with changes to the verifier or cryptographic signatures, because it targets BPF after those stages. There are potential mitigations, however. Bland suggested reserving memory ranges for BPF programs that don't overlap with the rest of kernel memory to make it harder for attackers to write to the pages while they're vulnerable. 

That idea isn't a complete solution, however, because it introduces a lot of complexity for memory management. Also, there's a limit to how much memory can be reserved for BPF. As with other proposals to increase security by carving up the kernel's memory, it can be difficult to judge what the correct size to allocate is. Bland did say that Mike Rapoport was working on [a change related to this](<https://lwn.net/Articles/933867/>). 

Bland summarized some related next steps for making "write then execute" scenarios harder to exploit, although not all of the proposals impacted BPF. Puranjay Mohan has [ a patch set](<https://lwn.net/ml/linux-kernel/20240324211518.93892-1-puranjay12@gmail.com/>) improving control-flow-integrity (CFI) protections on aarch64. Bland hopes to see LLVM's CFI hashing algorithm improved. Finally, there are plans to add more security monitoring for uses of the kernel's fixed map in EROFS. 

BPF's verifier already lets the kernel track many security properties, but now BPF developers are looking at what will be necessary to continue securing BPF programs both before the verifier (with signing) and afterward. Security is an ever-changing field; it seems likely that there will be more to report on all of these initiatives in time.
