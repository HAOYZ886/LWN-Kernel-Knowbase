---
title: Code signing for BPF programs
url: https://lwn.net/Articles/1017549/
date: "April 22, 2025"
category: "BPF-Security"
author: "By Daroc Alden April 22, 2025 LSFMM+BPF"
---

By **Daroc Alden**  
April 22, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The Linux kernel can be configured so that kernel modules must be signed or [ otherwise authenticated](<https://lwn.net/Articles/1012946/>) to be loaded into the kernel. Some BPF developers want that to be an option for BPF programs as well — after all, if those are going to run as part of the kernel, they should be subject to the same code-signing requirements. Blaise Boscaccy and Cong Wang presented two different visions for how BPF code signing could work at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit. 

#### The problem

Microsoft Azure has around 800,000 smart networking cards (NICs) that can run eBPF. It would be handy to be able to do live investigation of networking conditions on those NICs by using their BPF capabilities, Boscaccy explained. But Microsoft also has a policy that all kernel code running on its server infrastructure needs to be signed, as a way to prove that it was built on its build infrastructure from approved sources. 

Signing BPF code is not as straightforward as just signing the object file, however. BPF has a mechanism called [ "Compile Once - Run Everywhere"](<https://nakryiko.com/posts/bpf-core-reference-guide/>) (CO-RE) that introduces relocations to let the same program run on multiple different kernel versions. Unfortunately, right now CO-RE relocations are applied by user space, so by the time the kernel sees a BPF program, it has already been modified, rendering any digital signatures useless. 

[ ![\[Blaise Boscaccy\]](https://static.lwn.net/images/2025/blaise-boscaccy-lsfmmbpf-small.png) ](<https://lwn.net/Articles/1018062>)

Boscaccy had considered a few different ways to resolve the problem, including moving the BPF program loader (which performs relocations, sets up BPF maps, and performs related tasks) into the kernel, or [ an approach](<https://lwn.net/ml/bpf/20210423002646.35043-1-alexei.starovoitov@gmail.com/>) based on "light skeletons", which is a way for one BPF program to perform relocations for another. Moving the loader into the kernel is technically feasible, he said, and it makes signature verification trivial. The disadvantage is that it would reduce the flexibility of the current BPF ABI because user space would no longer be able to set up a BPF program as it pleases. Other BPF developers have also expressed disapproval of the idea. 

Light skeletons are an approach proposed by Alexei Starovoitov in 2021 to simplify BPF program loading; essentially, a light skeleton is a BPF program that runs in a restricted context and sets up the maps, global variables, and other state for another BPF program. The advantage is that because a light skeleton is much simpler than a general BPF program, the user-space loader can be minimal. Boscaccy's light-skeleton-based signing approach involved using [ fs-verity](<https://lwn.net/Articles/763729/>) to ensure that the simple user-space loader is the one desired. Then, using Linux security module (LSM) hooks, it ensures that the light skeleton only calls the functions necessary to relocate the BPF program. 

That approach turned out not to work for two reasons. Light skeletons leave the loaded BPF program in a modifiable state, so there was a potential time-of-check-to-time-of-use issue between when the skeleton finishes loading the program and when the program starts to run. Secondly, the root user could kill the parts of the solution that ran in user-space, which would disable verification. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

A systemd developer noted that systemd just crashes the system if something kills the necessary user-space components of the program, and thought that the same thing could work here. Boscaccy didn't like that answer, however. Starovoitov noted that trying to prevent root from breaking things is futile, because there are many ways that root can interfere with a running system already. 

#### Hornet LSM

In any event, Boscaccy's solution was to move the user-space components of his solution into the kernel. [ His solution](<https://lwn.net/Articles/1015105/>), called Hornet, is a full LSM that provides signature verification of light-skeleton-based BPF programs. It avoids the problems with his previous approach by hooking system calls to ensure that BPF programs aren't modified between being loaded by a signed program and executed. 

In his solution, the light-skeleton loader program is built as normal and then signed. Because the loader runs in a restricted environment, without access to many kernel functions, it doesn't need CO-RE relocations. Therefore, the signature remains valid and Hornet can verify it before permitting the program to run. Once the skeleton performs relocations for the real BPF program, Hornet can tell that the program should be permitted because it is being loaded from inside the kernel. The end result is that everything in the process is either done by the kernel (which is verified by secure boot), or by signed code. 

Starovoitov asked why Boscaccy created a new LSM, instead of adding this functionality to an existing LSM; Boscaccy replied that it could be added to an existing LSM in the future, but he wanted to focus on getting it working first. Wang asked whether this approach would work for programs like [ bpftrace](<https://bpftrace.org/>) that create new BPF programs in memory. Absolutely not, Boscaccy replied. The whole point of his work is to be able to deploy particular BPF programs without allowing people to put arbitrary code in the kernel. 

Daniel Borkmann asked whether signature verification could potentially be done from a BPF LSM, instead of an LSM written in C (as Hornet currently is). Boscaccy agreed that would be possible, but was worried about whether that would make it harder to see the work upstreamed. He said that he didn't want every distribution to be recreating its own independent solutions here. 

There was also the chicken-and-egg problem of how to load a BPF LSM that verifies signatures in a way that would have its signature verified. Ultimately, that approach would probably require building the BPF LSM into the kernel at compile time, which eliminates some of the benefit of writing the LSM in BPF. 

#### Two-phase BPF signing

Wang presented a different potential path for BPF signature verification: two-phase signatures. The problem with just signing BPF programs directly is that CO-RE relocations and other changes invalidate the signature, he reiterated. So why not sign the program both before and after the modifications? This would let the kernel continue to use existing user-space BPF-loading tools, and keep the signature verification logic reasonably simple. 

Wang's approach starts by making what he calls a baseline signature on the original BPF program, proving that it came from a trusted source. He compared this to having a document notarized. Then, a copy of libbpf verified with fs-verity or a similar solution would make the necessary changes to the program while loading it, and generate a signature for that modified program. Importantly, this second signature covers both the modified BPF program and the original signature, thereby establishing a chain of trust from the original builder of the BPF program, through the signed libbpf loader, to the final object presented to the kernel. 

[ ![\[Cong Wang\]](https://static.lwn.net/images/2025/cong-wang-lsfmmbpf-small.png) ](<https://lwn.net/Articles/1018062#anchor>)

The advantage of this approach is that it requires no kernel modifications — the signatures can be validated by a BPF LSM — and it is highly customizable by the user. Any modifications to a BPF program could work with this scheme, as long as the user had some way to trust the program making the changes. It also provides a precise log of what changes were made to the program at each stage of the process. 

The downside is that libbpf has to be trusted for this to work. It has to be trusted with a set of signing keys, even. Unlike Boscaccy's approach, it is trivial for root to interfere with the process. Managing the distribution of the signing keys is a challenge as well, although not one specific to this proposal, Wang noted. Also, the BPF LSM that verifies the signatures needs to be loaded as early in the boot process as possible. 

In order to mitigate these problems, Wang also wants to figure out how to build the needed BPF programs into the kernel. The main challenge that he identified was how to load a program into the kernel without using libbpf, but there is also the issue of what to do if the verifier fails to verify a BPF program that was built into the kernel. One potential way to handle that would be to use a user-mode helper to load and attach the BPF program from user space, panicking if it fails to verify. Starovoitov said that wouldn't be necessary, however: there are already some parts of the kernel that pre-load BPF programs, including [ bpffs](<https://qmonnet.github.io/whirl-offload/2023/11/04/dyk-bpffs/>), and Wang could reuse that approach. 

James Bottomley wondered how Wang proposed to secure the private key that libbpf would use to sign programs in this plan. He noted that [ dynamic kernel-module support](<https://lwn.net/Articles/24060/>) (DKMS) frequently uses a private key with a passphrase; if the user has to enter a passphrase for every loaded BPF program, that could be inconvenient. Wang thought that approach could work, though. Ultimately, it would be up to the user to choose how they wanted to distribute and protect the signing key. Bottomley suggested that perhaps the libbpf signing key could be made ephemeral, to ease distribution. Wang didn't see a problem with that, although he noted that he wasn't a security expert. 

Starovoitov wanted more details of what the second signature (the one made by libbpf) would cover. It covers the final form of the loaded BPF program, right before the [ `bpf()`](<https://www.man7.org/linux/man-pages/man2/bpf.2.html>) system call, as well as the original signature, Wang replied. One of the other audience members, however, raised an important point: if the signature is created after libbpf starts executing, but before the `bpf()` system call is made, then there's a large window of time where root could interfere with the process. Given that you can configure the kernel to require root to load BPF programs anyway, who does this scheme actually stop? 

After some discussion, the general consensus was that if libbpf is trusted, the second signature isn't really needed. On the other hand, if libbpf isn't trusted, the second signature isn't sufficient to establish a chain of trust. In both cases, root can definitely interfere with the process. 

There was also some question of how necessary either of the proposals actually were. I pointed out that if BPF can be loaded into the kernel at compile time, it didn't really seem necessary to have a complex signing process for distributing BPF programs. Starovoitov agreed, pointing out that one could embed a BPF program in a kernel module, and then sign and load that using the existing module-signing infrastructure. Boscaccy didn't like that idea, though, saying that this makes BPF programs ""honorary kernel modules, which sucks"". He went on to explain that he wanted some approach that gave the security benefits of using kernel modules, but without the overhead of managing lots of kernel modules. 

Bottomley pointed out that using kernel modules in this way loses one of the key benefits of BPF: portability between kernel versions. Boscaccy agreed. Andrii Nakryiko wasn't convinced this was a problem: embedding a BPF program into a kernel module still maintains source-level compatibility, the build system may just need to build several versions of the module. Bottomley thought that was still a big problem, because one frequently doesn't know that one will need a BPF program ahead of time. Starovoitov asked Boscaccy and Bottomley for some concrete examples of the kinds of BPF programs they were using; if they were using bpftrace or similar solutions, he agreed that the latency of building a new BPF program and loading it into the kernel would be important. 

At that point, the discussion devolved into a debate about the difference between security and provenance, and between communicating with user space and loading code into the kernel from user space. When time for the session ran out, it remained unclear whether there was enough support for Boscaccy's proposal, so this may be a topic that comes up again.
