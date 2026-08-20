---
title: SGX and security modules
url: https://lwn.net/Articles/798748/
date: "September 11, 2019"
category: "Security-Memory encryption"
author: "September 11, 2019 This article was contributed by Marta Rybczyńska"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

September 11, 2019

This article was contributed by Marta Rybczyńska

Software Guard Extensions (SGX) is a set of security-related instructions for Intel processors; it allows the creation of private regions of memory, called "enclaves". The aim of this feature is to work like an inverted sandbox: instead of protecting the system from malicious code, it protects an application from a compromised kernel hypervisor, or other application. Linux support for SGX has existed out-of-tree for years, and the effort of upstreaming it has reached an impressive [version 22 of the patch set](<https://lwn.net/ml/linux-kernel/20190903142655.21943-1-jarkko.sakkinen@linux.intel.com/>). During the upstreaming discussion, the kernel developers [discovered](<https://lwn.net/ml/linux-kernel/CALCETrV=wAsyWxtxQJ7y0xNrzkE863hTfU6Ysej48Gk9yPFJZw@mail.gmail.com/>) that the proposed SGX API did not play nicely with existing security mechanisms, including Linux security modules (LSMs). 

#### SGX and enclaves

SGX allows creation of memory regions that are expected to host a sensitive part of an application, such as the handling of private keys. Both the sensitive data and the code that manages it live within the enclave; except for defined entry points, they are inaccessible outside of it. In addition, the enclave code and data are encrypted in memory with ephemeral keys.

The enclaves [have their own page-protection regime](<https://lwn.net/ml/linux-kernel/20190903142655.21943-24-jarkko.sakkinen@linux.intel.com/>), as they are isolated from the rest of the system. The memory used by the enclave is handled separately from main system memory and the enclave must fit into a single contiguous linear virtual-memory range. With those constraints, loading the enclave requires specific preparations, including some assembly code. Interested readers can find an example [in the SGX self-test code](<https://lwn.net/ml/linux-kernel/20190903142655.21943-21-jarkko.sakkinen@linux.intel.com/>).

The enclave is managed, from the Linux side, by a device driver (LWN [looked into it](<https://lwn.net/Articles/786487/>) in April and, in 2016, [in the context of memory encryption](<https://lwn.net/Articles/686808/>)). It handles memory management within the enclave. The management API for SGX enclaves includes a mix of `ioctl()` calls to create an enclave, to add a page, and to initialize the enclave; it works along with with a few special SGX CPU instructions.

#### SGX and LSMs

In the discussion about the SGX security model, Greg Wettstein [pointed out](<https://lwn.net/ml/linux-kernel/20190418171059.GA20819@wind.enjellic.com/>) that the SGX API did not provide verification of the enclave code. He also mentioned an [attack [PDF]](<https://arxiv.org/pdf/1702.08719.pdf>) using SGX that allows malicious code running inside an enclave to steal data from an application. That makes the enclave code a possible security threat in its own right. 

Preventing such attacks requires that the kernel have control over which code can be run within an enclave. Andy Lutomirski pointed out that software verification is already available in Linux in the form of LSM policies and noted that there might be a bigger issue as well. While security policies in many systems do not allow applications to create executable memory regions with arbitrary content, the SGX API does allow exactly that. He wrote: 

There are many, many Linux systems that enforce a policy that *all* executable text needs to come from a verified source. On these systems, you can't mmap some writable memory, write to it, and then change it to executable. [...] Unless I'm missing it, the current SGX API is entirely incompatible with this model. 

This conclusion (reasonably) upset the kernel developers, adding to the list of known issues with SGX in general (the [SGX Wikipedia page](<https://en.wikipedia.org/wiki/Software_Guard_Extensions>) contains a list of other problems). Linus Torvalds [agreed](<https://lwn.net/ml/linux-kernel/CAHk-=wjmT=uC1=18ZYV1CMfP_FBUEjh9_rabH0g+a0z-L0cgHg@mail.gmail.com/>) that circumventing LSM policies was a fatal flaw: 

And if the current setup basically is "you have to disable reasonable SELinux protections that lots of distros use today", I think it's entirely reasonable saying "the downsides are bigger than the upsides". 

The consensus among the developers was that the control over enclave page loading must be given to the LSMs. An extensive [discussion](<https://lwn.net/ml/linux-security-module/CALCETrXf8mSK45h7sTK5Wf+pXLVn=Bjsc_RLpgO-h-qdzBRo5Q@mail.gmail.com/>) on the linux-security-module mailing list explored the interactions between the LSMs and SGX in great detail.

#### Details of the problem

In the SGX driver implementation, all enclaves are backed by the same device file (`/dev/sgx/enclave`). This allows SGX users to use ordinary system calls (including `mmap()` and `ioctl()`) to move pages to and from the SGX-specific hardware module called the "enclave page cache" (EPC). The pages in the EPC differ from typical pages because of the hardware restrictions: they are shared by processes at the hardware level and there is one-to-one mapping between virtual and physical addresses within an enclave. The enclaves also need specific SGX read, write, and execute permissions to the EPC pages.

Without the needed LSM hooks, and with SGX active, the LSMs are unable to apply policies with the granularity they need -- they could deny all access to SGX (by blocking access to `/dev/sgx/enclave`), but could not control the mapping of pages as writable and executable, or loading code from anonymous memory. As the enclaves are backed by the same file, any restrictions would apply to all enclaves.

The developers also found out that, because the SGX hardware bypasses the standard page permissions, user space could use SGX to gain access it could normally not have. The known case is the following: a hostile process can add a read-only page backed by a file lacking execute permission to the enclave, then call `mprotect()` to change the permissions in the enclave to gain execute rights on that memory. A solution to the issue is to require specifying enclave page protection in advance, allowing the kernel to apply its policies regarding executable pages within enclaves.

#### Toward a solution

Sean Christopherson entered the discussion with [a patch set](<https://lwn.net/ml/linux-kernel/20190531233159.30992-1-sean.j.christopherson@intel.com/>) attempting to reconcile SGX with LSMs. It included [an `mprotect()` hook](<https://lwn.net/ml/linux-kernel/20190531233159.30992-5-sean.j.christopherson@intel.com/>) in the `vm_ops` structure meant to be implemented by LSMs. The SGX code, when building an enclave, would call that hook to oversee the permissions for each page. This approach did not work, though, due to problems with auditing. The developers were unable to make sure that this hook would not generate false positives from the checks made at enclave loading time. The other option was to do checks at the time of `mprotect()`, but that could cause false alerts when the system configuration changes. For example, alerts could result when a file is moved between the loading of the enclave and the change of protection, or when the LSM policy is modified.

The [next version](<https://lwn.net/ml/linux-security-module/20190619222401.14942-1-sean.j.christopherson@intel.com>) tracked the page protection of the enclave separately from the virtual memory areas (VMAs) mapping the source files and from the hardware protections. The API requires user space to declare the protections of each page when it is added by specifying a combination of the `PROT_READ`, `PROT_WRITE`, and `PROT_EXEC` flags. Those flags define the maximal permissions that user space may request when mapping the page. As a result, when a process tries to `mmap()` or `mprotect()` on an enclave page, the call will fail if the protections in the VMA would be more permissive than the enclave page permissions. This tracking allows the SGX code to call into LSMs when the enclave is being built. The LSMs can then enforce their own policies on enclave pages.

During the discussion, Cedric Xing [sent](<https://lwn.net/ml/linux-security-module/cover.1561588012.git.cedric.xing@intel.com/>) a different patch set addressing the same issue. He used a different method: tracking the correspondence between the enclave pages and their origin (for example the files they were loaded from). That correspondence is kept in the LSM data structures.

#### The twist

All the proposed solutions were complicated, hindering their acceptance. A shift toward a consensus happened, though, when the developers came back to the basic requirements. Lutomirski [explained](<https://lwn.net/ml/linux-security-module/CALCETrXMAwHod_KZYPGWjTjg-fxOb1%3D02%3DQj2g1o624wOPfPZQ%40mail.gmail.com/>) that the only real requirement for SGX LSM support is to not allow the execution of arbitrary code, making the scope of the problem much simpler than what had been considered before. Jarkko Sakkinen, the original poster of the SGX patches, [proposed](<https://lwn.net/ml/linux-security-module/20190705160549.tzsck5ho5kvtdhit@linux.intel.com/>) deferring the complexity of loading the enclaves to user space and, in particular, splitting the roles of loading and running the enclaves. Finally, Stephen Smalley [summarized](<https://lwn.net/ml/linux-security-module/27e55a96-d5c4-69ed-a88d-7a3c26fb7f75@tycho.nsa.gov/>) an off-list discussion concluding that the solution only requires a decision of the policies to be used by SELinux for SGX. 

Christopherson noted that there is [still some work to do](<https://lwn.net/ml/linux-security-module/20190708172930.GA20791@linux.intel.com/>), but it can be done without affecting the SGX API, after the new code is upstreamed. The code will [have some drawbacks](<https://lwn.net/ml/linux-security-module/120cfcee-af82-83cd-fb73-a39ecddff2af@intel.com/>), as Xing explained, such as the inability to support self-modifying code in the enclaves.

#### Summary

This discussion took over two months and touched the low-level details of the kernel's memory-management and security primitives. The apparent solution is much simpler than what was proposed in the discussion, even though the exact details still need to be worked out. However, it seems that the developers can now concentrate on the remaining SGX upstreaming challenges.
