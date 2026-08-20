---
title: In search of a stable BPF verifier
url: https://lwn.net/Articles/1016853/
date: "April 14, 2025"
category: "BPF-Verifier"
author: "By Daroc Alden April 14, 2025 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Daroc Alden**  
April 14, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

BPF is, famously, not part of the kernel's promises of user-space stability. New kernels can and do break existing BPF programs; the BPF developers try to fix unintentional regressions as they happen, but the whole thing can be something of a bumpy ride for users trying to deploy BPF programs across multiple kernel versions. Shung-Hsi Yu and Daniel Xu had two different approaches to fixing the problem that they presented at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit. 

#### BPF in stable kernels

Yu presented remotely about the problem of running BPF programs on stable kernels. He began by recapping the process by which a patch ends up in a stable kernel: by including a "CC: stable@vger.kernel.org" tag in the patch, by being picked up by [ the AUTOSEL patch-selection process](<https://lwn.net/Articles/825536/>), or when the developer explicitly asks the stable team to include it. A patch that has been identified for inclusion in a stable kernel by one of those means needs to meet three additional criteria: the patch is present in mainline, the patch applies cleanly to the stable tree, and the stable tree builds after applying it. 

That whole process encodes some hidden assumptions, Yu said. For one, it assumes that a patch will work as-is when applied to an older code base, or at least that a patch which doesn't will be caught by the stable-release testers. But ""the elephant in the room is that it doesn't really work."" Patches taken from the current code base and applied to an older code base are not guaranteed to work. Patches sent to stable get less review, and stable-kernel testers do not systematically exercise BPF functionality. 

Yu proposed adding the stable trees to the BPF continuous-integration testing, starting with the 6.12 long-term-support kernel and continuing from there. That prospect isn't as simple as just running the tests on a new branch, however; if an error is found, where should it be sent? Yu listed three possibilities: the stable mailing list, a group of volunteers, or the BPF mailing list. Of the three options, he would most prefer to have a dedicated group of volunteers, but that may not be possible — it depends on whether anyone is willing to step up. 

Even if the stable tree is added to BPF's automated tests, the way the tests themselves are updated poses a problem. Usually, a fix and a test verifying the fix are submitted as part of the same patch set. If a fix is backported to a stable kernel, though, its accompanying test might not go back along with it. Yu spoke with Greg Kroah-Hartman, who agreed that he would accept BPF selftests into the stable trees, but he needs someone to identify them and ask for them to be backported. 

Finally, Yu said, not all fixes can be backported. That's fine — stable trees miss out on a lot of fixes in the name of stability — but it does pose a problem for security-relevant fixes. Changes to the BPF verifier often qualify as security-relevant. 

None of these problems are really BPF specific, Yu emphasized. Maybe there is some low-hanging fruit that can make the experience of using BPF on stable kernels less painful, though, and maybe the BPF developers can find someone willing to step up and take on that work. 

The assembled BPF developers pointed out some additional challenges that Yu might have overlooked, such as more load on the test machines and on the maintainers, but seemed generally in agreement that something needed to be done. 

#### Modularizing the BPF verifier

Xu had a different vision of how to enable stability for users of BPF. Xu's long-term goal is to ship BPF changes more quickly; the BPF subsystem changes quickly, but long-term support kernels stick around for a long time. The effort of trying to support BPF programs across a number of kernel versions in the face of that reality makes BPF painful to use, he said. His plan is to try to make the BPF subsystem into a kernel module that can be patched, shipped, and updated independently of the main kernel. 

That's a long and difficult project, however, so he would like to start with just the verifier. Separating the verifier out into a kernel module is a good place to start, Xu said, because it is already architecturally a pure function — that is, a function that transforms an input into an output without otherwise affecting the state of the system. The verifier takes in information about a BPF program, and outputs a judgment on whether it is safe, plus some information used by the just-in-time compiler — it doesn't call (many) other kernel functions. Also, changes to the verifier are one of the things that causes the most frustration for users. 

Xu put together [ a proof of concept](<https://lwn.net/ml/all/cover.1744169424.git.dxu@dxuuu.xyz/>), which he described as a pretty simple change, except for some complexity around supporting out-of-tree builds. From his testing, the modularized verifier seems to work. Currently, his proof of concept only encapsulates a single file: [ `verifier.c`](<https://elixir.bootlin.com/linux/v6.14/source/kernel/bpf/verifier.c>). He still thinks that's a useful starting point, however. 

[ ![\[Daniel Xu\]](https://static.lwn.net/images/2025/daniel-xu.png) ](<https://lwn.net/Articles/1017046/>)

Xu [ examined](<https://github.com/danobi/verifier-analysis/blob/master/analysis.ipynb>) the commits between kernel versions 6.3 and 6.13 to get some statistics. Most commits that touch `verifier.c`, don't change the rest of the kernel. These commits could, theoretically, be easily ported to a standalone stable verifier. Of the commits that affected only `verifier.c`, there were at least 73 bug fixes, based on the "Fixes" tags in the commits. 

So modularizing the verifier will, at least, allow users to receive some bug fixes on an otherwise unchanging kernel. The next question Xu attempted to address with his examination was: are there any other files that could be moved into his kernel module to further decrease the reliance on the rest of the kernel? To answer that question, he looked at files that were edited in the same commits as the verifier. 

The most commonly edited was [ `bpf_verifier.h`](<https://elixir.bootlin.com/linux/v6.14/source/include/linux/bpf_verifier.h>), which is ""conceptually private"", but in actuality is referenced by a few other files. After that, there was a long tail of other files that were less commonly modified. Xu admitted that this analysis wasn't as rigorous as it could be — for one thing, it should really consider patch sets as well as commits, since applying a single commit from a patch set is occasionally problematic — but he thought that this was still a useful exercise. 

For the next steps, Xu wants to move the code to build the modularized verifier out of tree and add continuous-integration testing across many different kernel versions. From his explanation, I believe his intended workflow is for the BPF developers to continue maintaining the BPF verifier in-tree, with the out-of-tree version functioning similarly to stable kernels: as a more stable alternative that still receives cherry-picked bug fixes. 

If that goes well, and the modularized verifier is actually helpful, he plans to look at implementing a well-defined interface to make it easier to maintain the verifier out-of-tree. He also intends to modularize more components of the BPF subsystem, work with distributions on distributing appropriate versions of the verifier, and eventually support running the verifier in user space as part of compilers targeting BPF. 

Someone in the audience asked why, if Xu wanted to make a version of the verifier that could run across multiple kernels, he didn't use BPF's [ struct_ops mechanism](<https://lwn.net/Articles/811631/>). Xu replied that it was a cool idea, but that the verifier probably executes a lot more than a million instructions (the current limit for BPF programs). Daniel Borkmann wanted to know whether the verifier was really what was giving users who need to support multiple kernel versions problems — isn't the changing set of kernel functions available to be called by BPF programs a bigger problem? Xu didn't think so, pointing out that functions are either there or they're not, so dealing with missing functions is relatively straightforward. But currently he is dealing with ""a bunch"" of different verifiers in production, and would really like to get it down to just two or three. 

Eduard Zingerman asked whether Xu expected to be able to reduce the entanglement between the existing verifier code and the rest of the kernel further; Xu thought that it was probably possible, and shared a list of kernel symbols that the verifier currently refers to for the assembled developers to ponder. Another person wanted to know whether a modularized verifier would allow for more thorough testing. Xu thought so. Fuzz testing, especially, would be easier without having to run the whole kernel. BPF development would also be nicer if tweaking the verifier did not require rebuilding the whole kernel. 

Alexei Starovoitov asked whether Xu had thought about how to handle changes to the verifier that actually change the memory layout of the verifier state. Xu replied that it would need to be addressed on a case-by-case basis. Other developers expressed skepticism that a modular verifier would actually be helpful to users, given that users often need to wait on new kernel functions to become available to BPF programs. David Faust, on the other hand, was enthusiastic about the idea of being able to run the verifier code in user space, since he has repeatedly asked for a way to let GCC verify the BPF code it generates. 

Whether the modular verifier will be adopted remains to be seen — and many BPF developers are skeptical — but Xu's working proof of concept suggests that it may not be as daunting a task as it first appears.
