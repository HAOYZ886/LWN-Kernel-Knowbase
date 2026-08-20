---
title: "Multiple memory classes for address-space isolation"
url: https://lwn.net/Articles/1014440/
date: "March 21, 2025"
category: "Memory management-Address-space isolation"
author: "By Daroc Alden March 21, 2025"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Daroc Alden**  
March 21, 2025

Brendan Jackman has been working to try to get ahead of the next hardware CPU vulnerability before it gets discovered. In January, he posted the second version of [ a patch set](<https://lwn.net/ml/all/20250110-asi-rfc-v2-v2-0-8419288bc805@google.com/>) that introduces [ address-space isolation](<https://lwn.net/Articles/974390/>) (ASI) as a way of preventing future CPU vulnerabilities from leaking important information. The core concept is to ensure that data that is not currently needed is not present in memory, so that speculative execution cannot leak it. The work is nowhere near ready to be incorporated into the mainline kernel — not least of all because it has a large performance impact in its current form — but it is likely to once again be a topic of discussion at the 2025 [ Linux Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). 

Jackman's patch set introduces different classes of memory. The classes are effectively isolated from one another, in order to avoid leaking information between them. [ The first version](<https://lwn.net/ml/all/20240712-asi-rfc-24-v1-0-144b319a40d8@google.com/>) only differentiated between memory mappings for KVM virtual machines and everything else, but reviewers wanted a demonstration that his approach could handle more than just two classes. Therefore, the most recent version of the patch set has a separate class for kernel code that handles certain system calls that don't require access to sensitive data as well. Each class of memory has its own independent address space, intended to contain only information relevant to a particular part of the system. When the kernel needs to access user-space memory, or run code for a virtual machine, it needs to switch to using that class (and that address space). Speculative execution cannot leak information that is not mapped into memory, and the system keeps track of switches between classes in order to flush any other microarchitectural state that could be used to leak information. 

If this idea sounds similar to [kernel page-table isolation](<https://lwn.net/Articles/741878/>) (KPTI), that's because it is. KPTI ensures that kernel and user space have almost entirely separate address spaces, with separate page tables. This prevents user-space code from using speculative execution to leak memory from the kernel. ASI is essentially taking the idea one step further, by carving up the kernel into (for now) separate sections for KVM operations, the parts needed for responding to certain non-sensitive system calls, and everything else. Somewhat confusingly, the patch set calls the address space for responding to system calls that don't need sensitive information the user-space ASI class, even though it is used in the kernel, not in user space. 

Jackman's patch set [ does not interfere](<https://lwn.net/ml/all/20250110-asi-rfc-v2-v2-3-8419288bc805@google.com/>) with KPTI; systems where both KPTI and ASI run together will currently end up with four separate address spaces per task: the unrestricted kernel address space, the user address space (added by KPTI), and one restricted kernel address space for each ASI class (KVM and ASI's user-space class). There are some fundamental drawbacks to this proliferation of address spaces, however. 

Ideally, this design would make it trivial to respond to the next discovered CPU vulnerability; in the most likely case of another speculative-execution-based vulnerability, the kernel would already be protected. If some other way of leaking information with CPU state is discovered, kernel developers would just have to make sure that the relevant CPU state was included in the list of things to flush when changing classes. That list currently includes the translation lookaside buffer, the level 1 CPU cache, and branch-predictor state. Leaking information within a class are not a concern, because the idea is to only put the minimum necessary information in each class, so code running in an address space should already have access to it anyway. 

As ever, the problem is performance. The impact of the patches varies greatly between different workloads, but the greatest impact that Jackman observed during testing was a 70% reduction in throughput on a particular disk I/O benchmark. Because each class of memory needs to have its own, separate set of mapped pages, the patch set removes the page cache from the kernel's direct map. File operations — which almost always need to touch the page cache — require changing ASI class for each operation, which is slow. Theoretically, such a change isn't necessary, if non-sensitive parts of the page cache can be added to the relevant ASI class. This is one of several potential improvements that Jackson called out in his commit messages. He hopes to implement them in the future, but right now, he's focused on validating the approach and seeing whether the kernel development community would support implementing it. 

Borislav Petkov [ thought](<https://lwn.net/ml/all/20250219105503.GKZ7W4h6QW1CNj48U9@fat_crate.local/>) that Jackman's patch set presented a ""weird API"". It adds three different functions that parts of the kernel can use to manipulate the current ASI class: `asi_enter()`, which enters a restricted class; `asi_relax()`, which signals the end of untrusted code; and `asi_exit()`, which does the actual work of exiting a restricted class. Petkov thought the naming was unintuitive — calls to `asi_enter()` don't necessarily match one-to-one with calls to `asi_exit()`. 

The design, Jackman [ said](<https://lwn.net/ml/all/CA+i-1C3Ei=dNRUaHThaBr3hPy8=5O+ocRHgw=YLkX6goJWw4ow@mail.gmail.com/>), was something of a holdover from when he had only implemented support for KVM. Calls to `asi_enter()` are actually balanced with calls to `asi_relax()`, as he explained with this pseudocode description of that support: 

```
ioctl(KVM_RUN) {
            enter_from_user_mode();
            while !need_userspace_handling() {
                asi_enter();
                vmenter();
                asi_relax();
            }
            asi_exit();
            exit_to_user_mode();
        }
```

Since `asi_exit()` does the expensive work of leaving a restricted ASI class, arranging the API so that the kernel needs to call it as little as possible is necessary for performance. Petkov [ thought](<https://lwn.net/ml/all/20250227120607.GPZ8BVL2762we1j3uE@fat_crate.local/>) it would make more sense to ensure that the functions that need to be balanced with each other have corresponding names, and suggested renaming `asi_exit()` to `asi_switch()` and `asi_relax()` to `asi_exit()`. 

Jackman was [ receptive](<https://lwn.net/ml/all/20250228084355.2061899-1-jackmanb@google.com/>) to the criticism, but pointed out that there were other complications: eventually, he would like to completely remove `asi_exit()`, and have the kernel automatically switch ASI classes when a need is detected at run time. This could potentially be another way to avoid unnecessary ASI class switches, for better performance. So any API design should make it easy to eventually do away with whatever equivalent of `asi_exit()` the kernel developers end up committing to. Also, the API needs to ensure that there's no period during which an interrupt can arrive and accidentally return to code that should run in a restricted ASI class without applying the restriction. 

The second concern [ didn't make sense](<https://lwn.net/ml/all/20250314131419.GJZ9Qrq8scAtDyBUcg@fat_crate.local/>) to Petkov, but Junaid Shahid [ shared](<https://lwn.net/ml/all/5aa114f7-3efb-4dab-8579-cb9af4abd3c0@google.com/>) a simpler example that [ made it click](<https://lwn.net/ml/all/20250315123621.GCZ9V0RWGFapbQNL1w@fat_crate.local/>). Petkov suggested an API that separates out the operation of telling interrupt handlers that they may need to worry about returning to a restricted address space, actually switching to that space, and tracking whether the CPU was currently executing restricted code. 

Jackman [didn't see](<https://lwn.net/ml/all/Z9gKLdNm9p6qGACS@google.com/>) why updating interrupt handlers and switching address spaces needed to be separate operations, and proposed using functions called `asi_start()`/`asi_end()` to handle those and functions called `asi_start_critical()`/`asi_end_critical()` to demarcate untrusted code. [ Shahid](<https://lwn.net/ml/all/14f9106d-3a34-4f10-ba4e-465c73c94eba@google.com/>) and [ Petkov](<https://lwn.net/ml/all/20250319152334.GLZ9rhdrBYW2yXRbY3@fat_crate.local/>) were generally happy with that design. 

At the time of writing, the review and discussion are ongoing. While Jackman's updated patches seem to demonstrate that ASI could flexibly handle multiple different classes of memory, there are still a lot of questions that need to be answered before the work will be ready. Getting ahead of hardware vulnerabilities with a generic protection like the one Jackman proposes is a daunting task, but a worthy one. It remains to be seen whether other kernel developers will agree with this approach.
