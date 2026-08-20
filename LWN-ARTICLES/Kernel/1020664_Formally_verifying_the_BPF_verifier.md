---
title: Formally verifying the BPF verifier
url: https://lwn.net/Articles/1020664/
date: "May 23, 2025"
category: "BPF-Verifier"
author: "By Daroc Alden May 23, 2025 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Daroc Alden**  
May 23, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The BPF verifier is an increasingly complex and security-critical piece of code. When the kinds of people who are apt to work on BPF see a situation like that, they naturally question whether it's possible to use formal verification to ensure that the implementation of the code in question is correct. Santosh Nagarakatte led the first of two extra-long sessions in the BPF track of the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit about his team's work formally verifying the BPF verifier with a custom tool called [ Agni](<https://github.com/bpfverif/agni>). 

#### Agni's history

[ ![\[Santosh Nagarakatte\]](https://static.lwn.net/images/2025/santosh-nagarakatte-lsfsmmbpf-small.png) ](<https://lwn.net/Articles/1021820>)

Work on Agni began about 6 years ago, Nagarakatte said, when he got interested in the [ PREVAIL BPF verifier](<https://vbpf.github.io/>), and met other people excited to study it. Since then, Harishankar Vishwanathan, Matan Shachnai, Srinivas Narayana, and Nagarakatte have been working at the [ Rutgers Architecture and Programming Languages Research Group](<https://people.cs.rutgers.edu/~sn349/rapl/index.html>) to develop the tool. 

The Linux kernel's BPF verifier is probably the first real instance of formal verification ""in production"", Nagarakatte said. Other projects that use formal verification tend to do so ""on the side"", not as part of the running, deployed system. That makes it interesting because writing correct formal verifiers is hard, and the BPF verifier will often be running in a context where it's hard for the original developer to spot errors. 

So, he asked, can we understand the algorithms that the BPF verifier uses, and guarantee that they're correct? The BPF verifier has a lot of different components, so Nagarakatte and his team decided to start by tackling value tracking: the part of the verifier that determines what values a variable can have at different points in the program. Narayana's later session, which will be the subject of a separate article, covered their subsequent work on checking whether the verifier's path-pruning algorithm is correct. 

Their first stab at the problem was to manually encode and check some proofs about the BPF verifier's abstract-value-tracking implementation. That worked fine for addition, but they couldn't make it work for the verifier's checking of multiplications. As a result of that experience, they ended up writing a new algorithm for multiplying abstract values that was amenable to verification, and got that accepted into the mainline kernel. So, from that work, they were confident that addition and multiplication were correct, which is already useful. 

The BPF verifier changes all of the time, however, and manually keeping their proofs up to date was clearly not going to be feasible. That's where Agni steps in: it takes the C source code of the BPF verifier and converts it into a [ satisfiability modulo theory](<https://en.wikipedia.org/wiki/Satisfiability_modulo_theories>) (SMT) problem that can be automatically proved or disproved by an [ SMT-LIB](<https://smt-lib.org/>) implementation such as [ Z3](<https://github.com/Z3Prover/z3?tab=readme-ov-file#z3>). If the solver can prove that the verifier is correct, that's excellent. 

If it finds a counterexample, however, the raw output is not particularly useful. Ideally, Nagarakatte's team wants the BPF developers to be able to use Agni as an extra check during development — something that can be used to test changes before they actually make it into the kernel. In pursuit of that goal, they added a program-synthesis component. If the SMT solver finds that the verifier is not correct, Agni will take the output of the SMT solver and use it construct a proof-of-concept BPF program that triggers the bug in the verifier. That can be fed back to the developer to illustrate where the failure comes from. 

#### Verifying arithmetic

With that high-level history of the project out of the way, Nagarakatte went on to explain how Agni actually does this. First, it takes the C source code and compiles it to LLVM's intermediate representation (IR). Agni doesn't need to handle every corner-case of the IR because it turns out that the verifier's code is not ""as bad as other real world C"" — it uses a fairly limited subset of the language. Once Agni has the IR, it uses LLVM's dead-code elimination to focus on a single operator at a time by discarding all of the parts of the verifier that aren't relevant to that operator. 

Those operators are used to combine the verifier's abstract representations of what a variable _could_ be. So it's not as simple as adding two concrete numbers — instead, the verifier has to be able to answer questions like "if register 1 has a number between 0 and 100, and register 2 has a number between 3 and 5, is their sum less than the length of this array?". This information is used throughout the verifier to ensure that accesses are within-bounds and aligned. 

In particular, the verifier tracks which bits of a value are known exactly, as well as what its range of possible values is as a signed or unsigned number. Shung-Hsi Yu led a session at the 2024 summit about [his work simplifying the representation of these abstract values](<https://lwn.net/Articles/977815/>). 

For each mathematical and bitwise operator, Agni takes the LLVM IR and translates it into a machine-checkable specification that the operator is implemented correctly. This transformation ends up using type information from the LLVM IR, which poses a problem because some of that type information is not available in LLVM version 15 or higher. Eventually, when the kernel updates to require LLVM 15, Agni will break and the BPF developers will need to find an alternate approach. That was a problem Nagarakatte wanted to discuss with the assembled developers in more depth. 

What it means for an abstract operator of this type to be correct ("sound") is remarkably straightforward, as complicated mathematical definitions go. Suppose that there are two abstract values (considered as sets of possible values, even though this isn't how the verifier represents them in memory), P and Q, and two specific numbers, x and y, which are members of P and Q respectively. The verifier's implementation of "+" is sound if the abstract representation that comes out of calculating the operation of "+" on two registers containing P and Q always contains the number "x + y". That is to say, given some specific numbers that are correctly modeled by two abstract register states, adding the two numbers should produce something that is correctly modeled by the addition of the two abstract states. 

#### Complications

At first, they had planned to verify each way that the verifier tracks values (as known bits, and signed and unsigned ranges) independently. That turns out not to work, however, because the verifier actually shares information between these representations. For example, if it knows that all of the bits other than the least significant two are zero, it also knows that the signed and unsigned ranges are 0-3. In the absence of this sharing of information, the BPF verifier's implementation would be unsound. The academic term for this sort of thing is a "shared refinement operator"; a refinement operator being something that slims-down an abstract value by ruling out impossible values. 

Once they were able to successfully model the shared refinement operator, they finally got confirmation that modern kernels are sound. Specifically, they were able to show that kernels from version 5.13 onward were sound. The oldest kernel version they tested was 4.14, so that left the problem of how to demonstrate an actual problem in the kernels between those versions — or, if they could not, to discover another deficiency in Agni. 

This is where the idea of synthesizing BPF programs came in. If Agni can prove that the verifier's implementation of an operator is not correct, that essentially means that it has figured out a way to add two registers that outputs a concrete value the verifier is not expecting. Then the problem becomes: how to create a BPF program that puts the verifier into those specific abstract states, and ends up calculating the bad final value. 

They saw that the real-world failures from earlier kernel versions were generally caused by fairly simple conditions, and so ultimately selected a brute-force approach. Agni will consider every BPF program that uses a series of arithmetic instructions ending in the flawed one in increasing order of program length, and return the smallest that triggers the bug. 

This approach worked to generate several proof-of-concept BPF programs for older kernels. Unfortunately, SMT-solving is NP-complete, and, as the verifier has become more complicated, the time it takes Agni to verify that its implementation is correct has grown. Agni ran against kernel version 4.14 for 2.5 hours, against version 5.13 for ten, and against version 6.4 for several weeks. Then, Andrii Nakryiko posted [ a patch](<https://lwn.net/ml/all/20231027181346.4019398-1-andrii@kernel.org/>) that improves the accuracy of the verifier's shared refinement operator, which significantly slows Agni's analysis, leading to timeouts. 

#### Going faster

At this point, the team working on Agni was in a rough place: they had a working tool that could turn up real bugs in the BPF verifier, but it wasn't going to be able to keep up with new kernels because of scaling problems. So they decided to try to break the problem down into subproblems that could be solved independently. 

Each abstract operator that Agni extracted from the verifier came to about 5,000 lines of SMT-LIB code. Of those, about 700 lines are the actual operator itself, and the rest is the code for the shared refinement operator. They decided to see if they could verify the shared refinement operator _once_ , and share that proof between all of the operators. 

That approach didn't work, because it turns out that the shared refinement operator was also masking latent unsoundness in some of the bitwise operations. These didn't represent real bugs, because in the actual verifier the shared refinement operator was always used. But they did represent a barrier to Agni, because it seemingly made it impossible to verify the shared refinement operator independently of the operations that used it. 

The solution ended up being to submit a small fix for the bitwise operators. Once those patches were accepted, the divide-and-conquer approach became feasible, and Agni's run time dropped to less than 30 minutes for kernel version 6.8. 

#### Future work

John Fastabend asked whether modeling the shared refinement operator separately allowed them to say whether the fixed versions of the bitwise operators were more or less precise (in the sense of more closely approximating the minimal set of possible values of the output). Nagarakatte said that is was exactly as precise, actually. Daniel Borkmann asked whether they had looked into whether the shared refinement operator could be made more precise. Nagarakatte said that they were experimenting with that internally, and once they have a better refinement operator that they're confident won't break anything, they'll submit a patch set. 

Fastabend asked whether they would be able to use the tool to find redundancy in the C code — that is, conditions that the verifier checks even though a check is not needed. Nagarakatte responded that one of his students was working on a project to synthesize abstract operators from scratch, which ""should be as good or better than what the kernel does"". They've already come up with a more concise representation for abstract values, although the data structure the kernel uses has already been proved to be maximally precise. 

Recently, Nagarakatte's student shared a patch that improves the precision of the multiply instruction to work better with negative values. He wants to work with them to put together a paper on the technique once they can explain it, at which point it may be applicable to other parts of the verifier. 

With Agni fully described, he then wanted to turn to the topic of how to move forward. The main upcoming problem Nagarakatte foresees is the kernel moving to LLVM 15\. His preferred resolution would be for the BPF developers to rewrite the verifier in some abstract specification language, which could be used as an input to Agni and as a source of generated C code. He was optimistic that writing the verifier in a higher-level language would make improving the verifier and reviewing it easier for everyone. 

Borkmann mentioned that Nagarakatte had proposed the idea of embedding some kind of domain-specific language (DSL) for the verifier in the comments of the C source code; he asked whether that invites the problem of ensuring that the DSL actually corresponds to the C code. Nagarakatte agreed that was a problem, but it's a much easier problem than parsing C source code correctly without LLVM. 

Another audience member pointed out that any DSL for verifier code would be yet another language to learn — ""how do we make that easy?"" Nagarakatte explained that when he said it would be nice to use a DSL, he didn't mean anything too complicated. One of the problems that they're dealing with in Agni is handling arguments that are passed in pointers; right now, they're relying on LLVM's analysis to remove memory accesses from the code to make modeling it easier. If the developers could specify argument types with a DSL, it could potentially simplify things. 

One person asked whether this kind of approach could be extended to other parts of the kernel. Nagarakatte said that there are other static-analysis-based approaches that could be applied to other parts of the kernel. The [ seL4 microkernel](<https://sel4.systems/>), for example, has a formal proof of correctness. He hasn't been working on that, though; he has been focusing on Agni. Ultimately, as with so many things in open source, it just needs someone to take the time to make it happen. 

Amery Hung wanted to know whether there were other parts of the verifier that could be formally verified, beyond arithmetic operations. Nagarakatte said that he was excited about looking at Spectre mitigations, which he thinks may be provably unnecessary in some places. The group is also planning to look at improving precision, and verifying the correctness of the verifier's path-pruning algorithm, which was the subject of Narayana's talk. The path-pruning logic is ""leaving a lot on the table"", he said, because the logic is widely dispersed throughout the code, which makes it hard to simplify. There were a few more minutes of clarification about the exact claims that Agni proves, and why newer LLVM versions were problematic, but eventually the session came to a close.
