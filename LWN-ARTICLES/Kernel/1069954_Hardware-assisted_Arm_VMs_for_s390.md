---
title: "Hardware-assisted Arm VMs for s390"
url: https://lwn.net/Articles/1069954/
date: "May 5, 2026"
category: "Architectures-Arm; Architectures-s390; KVM"
author: "By Daroc Alden May 5, 2026"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Daroc Alden**  
May 5, 2026

A recent [ patch set](<https://lwn.net/ml/all/20260402042125.3948963-1-seiden@linux.ibm.com/>) from Steffen Eiden and others has set the groundwork for allowing hardware-assisted emulation of Arm CPUs on s390 CPUs. [ Version two](<https://lwn.net/ml/all/20260428155622.1361364-1-seiden@linux.ibm.com/>) of the posting fixes a handful of smaller problems, but does not differ much. The patches were welcomed by the Arm maintainers, pending some discussion of how the collaboration between the architectures could be structured to prevent maintainability problems on the Arm side. When those details are resolved, the patches could pave the way for transparently running Arm-based virtual machines (VMs) on s390 hosts at native or near-native speeds. 

The core of the feature is [ a patch](<https://lwn.net/ml/all/20260402042125.3948963-20-seiden@linux.ibm.com/>) that adds support for a new s390 instruction called "Start Arm Execution" (SAE). It performs a similar function to the existing "[Start Interpretive Execution](<https://www.qemu.org/docs/master/system/s390x/vfio-ap.html#start-interpretive-execution-sie-instruction>)" instruction on s390 that is used to enter a hardware-assisted virtual machine while keeping the virtual CPU state separate from the host CPU. Both instructions take a pointer to a "control block" that describes how the virtual CPU should be set up and entered. The difference is that a SAE instruction's control block sets the instruction pointer to a block of memory containing Arm instructions and interprets them as such, rather than s390 instructions. 

In theory, this allows someone with a s390 CPU new enough to support the feature to run an Arm virtual machine directly. There clearly has to be a translation from Arm machine code to s390 machine code at some point, but the CPU handles that internally. How exactly it does that is not clear from the patch set. 

On the kernel side, while the idea is simple, the implementation is a bit more complex. When virtual machines make calls to their hypervisor, they do so using an architecture-specific interface. If Arm virtual machines are to run on s390 unmodified, the KVM code in an s390 kernel needs to be able to interpret Arm hypercalls. To support that, Eiden's patch set moves the interface definition and related header files to `include/uapi/arch/arm64/`, allowing other architectures to reference them. Amusingly, this allowed some duplicated code to be cleaned up; the patches ended up removing more lines of code than they added. 

There is additional work to be done once these patches are merged, however. The s390 architecture is also gaining instructions for manipulating Arm registers, handling interrupts, and so on. Eiden hopes to add support for those over the coming months in conjunction with the rest of the s390 developers. 

There was relatively little feedback on the patch set, but Marc Zyngier had [a lengthy response](<https://lwn.net/ml/all/86o6jd2925.wl-maz@kernel.org/>) written in consultation with Will Deacon. They were supportive of the effort, but worried that moving some of the Arm-specific code to a shared directory was messy and could interfere with the code's maintainability. They suggested that using symbolic links, relative paths, or code generation could allow the existing Arm code to remain it its accustomed location without unduly restricting s390 code. 

Eiden's [ reply](<https://lwn.net/ml/all/20260423122549.361343-A-seiden@linux.ibm.com/>) explained that they had tried using symbolic links for header files originally, but feared the approach wouldn't gain support since it was somewhat messy. Some of the code is also automatically generated, reusing the Makefile rules from the Arm code. But he agreed to prototype an alternative that makes use of symbolic links and post that for comparison. That patch set was [ sent on the April 28](<https://lwn.net/ml/all/20260428160527.1378085-1-seiden@linux.ibm.com/>) and has not received any review as of the time of writing. 

A more serious concern was how s390 would keep up with changes to the Arm virtual-machine-guest API, which is still evolving, Zyngier said. For example, there are some CPU-vulnerability mitigations that require cooperation from guests. When a new one is discovered, Arm assigns a hypercall function number for the new interface needed, and KVM implements it. The s390 code will presumably need something similar, both adding stubs for new Arm-specific mitigations and working with Arm to add any s390-specific interfaces needed. Those should be limited to mitigations, however, since the Arm KVM code ""makes a point of forbidding any use of implementation specific instruction or system registers,"" and Zyngier expects the s390 code to do the same. 

Eiden was not overly worried about this, saying that a primary goal of the project was to be able to run unmodified Arm guests, and so he and his fellow s390 developers would be treating the Arm code as the source of truth. As such, they had no plans to introduce s390-specific features or abrogate the usual process. 

Zyngier and the other Arm maintainers are, understandably, not familiar with the details of s390. So, Zyngier and Deacon also asked for help with documentation, testing, and debugging, to ensure that changes to the Arm code do not have adverse effects on s390. 

> Finally, we feel it would be beneficial for both projects to swap prisoners and have cross-reviewers in MAINTAINERS, so that there is an s390 reviewer added to KVM/arm64, and an arm64 reviewer added to KVM/s390. 

Eiden readily agreed, and suggested that cross-compilation of the kernel for s390 was a good starting point for testing. He offered access to s390 virtual machines hosted by IBM for doing native builds, as well. Eiden plans to add Arm's continuous-integration-testing branches to s390's build infrastructure, to catch any breakage early. He also thought an exchange of maintainers made sense, and [ added himself](<https://lwn.net/ml/all/20260428155622.1361364-15-seiden@linux.ibm.com/>) as an Arm KVM reviewer in the second revision of the patch series. 

Eiden and Zyngier made plans to meet up with the other s390 and Arm kernel developers at the Linux Plumbers Conference ~~Linux Storage, Filesystem, Memory Management, and BPF Summit~~. Hopefully an in-person meeting will suffice to figure out any remaining details, but since everyone seems happy enough with the change that is quite likely. 

[ Thanks to Andi Holmes for bringing this topic to our attention. ]
