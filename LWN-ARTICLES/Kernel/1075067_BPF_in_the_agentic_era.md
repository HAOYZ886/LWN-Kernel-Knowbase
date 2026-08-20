---
title: BPF in the agentic era
url: https://lwn.net/Articles/1075067/
date: "June 3, 2026"
category: BPF
author: "By Daroc Alden June 3, 2026 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Daroc Alden**  
June 3, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Alexei Starovoitov gave ""less of a presentation, more of a scream of realization"" at the BPF track of the 2026 [ Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). He shared a set of ideas for how BPF could change to avoid being swept away by the sea-change in programming represented by modern large language models (LLMs) and the coding agents based on them. In a follow-up session, the discussion covered more problems with how coding agents use tools like bpftrace, and the current deluge of patches in need of review in the BPF subsystem. 

He wanted to be clear that these ideas were not something he was unilaterally deciding on, they were ""just looking at how the world has to be"". Coding agents do their best work when given tight feedback loops, he said. They write code, see errors, and make fixes. This makes them good at handling tedious jobs such as large refactoring operations. That feedback loop doesn't work well in the BPF ecosystem, especially when one has to boot a virtual machine in order to properly test a change. 

[ ![\[Alexei Starovoitov\]](https://static.lwn.net/images/2026/alexei-starovoitov-lsfmmbpf-small.png) ](<https://lwn.net/Articles/1075340>)

Even when a BPF programmer has attempted to load their program into the kernel, errors from the verifier ""are insane"" — huge dumps of data that obscure the actual source of the problem. In Starovoitov's experience, writing BPF code is the only case he has observed where LLMs will give up rather than producing _something_. 

BPF can never crash the kernel. ""That's what BPF stands for."" That is the one thing that makes the ecosystem unique. So, the BPF verifier isn't going away, and it needs to run in the kernel because user space is untrustworthy. This adds a certain amount of unavoidable latency and complexity between writing a BPF program and getting feedback on it. Given that constraint, how can BPF developers shorten the feedback loops between writing a BPF program and getting a useful error message? 

Right now, the verifier essentially has two roles: finding places where the programmer made a mistake, and acting as a security boundary. The first part doesn't need to be done in the kernel; languages such as Rust do a great job of confronting programmers with their mistakes in user space. The BPF developers should be working to make BPF easy to use with Rust, so that simple mistakes can be caught quickly, Starovoitov said. Any program that does pass the Rust compiler — and doesn't do anything obnoxious with unsafe code or inline assembly — should pass the verifier as well. Obviously programmers will still be able to cause verifier errors if they try, but well-intentioned, normal BPF programs written in Rust should not cause verifier errors, he said. 

Making this happen may require further work on the verifier, or the careful addition of run-time checks to BPF. But it's a goal that Starovoitov believes is possible and worthwhile. There are other ways to shorten the feedback loop for BPF programmers, of course. If BPF could be made to work in [ user-mode Linux](<https://en.wikipedia.org/wiki/User-mode_Linux>), for example, there would be no more need to test BPF code in virtual machines. 

This confused one member of the audience, who objected that Starovoitov had previously soundly rejected the idea of moving the BPF verifier to user space. The real kernel can't trust user space, Starovoitov said, but it should be possible to run the verifier in user-mode Linux in order to shorten the time it takes to receive a verifier error if one is forthcoming. 

All of these changes are coming quickly. In a few months, ""there will be no fighting the verifier anymore,"" Starovoitov promised, as long as one writes BPF programs in Rust. The verifier will use run-time checks where necessary to make everything work. José Marchesi asked whether that change was currently in development. Starovoitov confirmed that it was. 

Daniel Borkmann asked how the output from the verifier would change. Current verifier errors are far too opaque, Starovoitov said. They say things like "register is not init" — what is a programmer supposed to do with that? The verifier needs error messages that say what happened and how to fix it. This is another area where Starovoitov wants to take inspiration from Rust, and, in particular, the formatting of its error messages. ""You guys are all dinosaurs; Rust won as a language,"" he said, at least in part because of the quality of its error messages. 

In the end, he wants there to be almost no verifier errors. The ones that remain should include details about what went wrong and how to fix it. This change isn't going to happen overnight, he said, but ""it is where we have to go"". 

There was some discussion about even bolder proposals, such as integrating WebAssembly into the kernel. Starovoitov was supportive of the idea if it could be made to work, but didn't think that was likely. When I spoke to him after the session, he said that he thought previous attempts to bring WebAssembly to the kernel were too slow for serious use, but that if someone was willing to do the work to bring it up to par with the existing BPF JIT implementation, they should do so. 

At that point, Starovoitov dove into the details of how his suggested transformation could be achieved. There are a number of limitations of the verifier that must be removed, he said. These include the inability to handle functions with more than six arguments, indirect function calls, returning structures by value, support for 128-bit integers, larger BPF program stacks, deeper call chains, removing the one-million-instruction limit, and long jumps in BPF programs. Also, the BPF developers will need to remove the discrepancies between the verification of static and global functions. ""Of course, this helps humans too."" 

Some of these things, such as larger program stacks, have already had the groundwork laid by previous improvements to BPF. For the rest, Starovoitov hopes to move more data into [ BPF arenas](<https://lwn.net/Articles/961941/>), which have one flat address space. Once that is done, a "global" memory allocator (really one per arena) makes many of the other problems significantly simpler. That plan does pose problems for Rust code that uses virtual dispatch tables; currently, the Rust compiler emits these tables intermixed with the code, which BPF is not set up to handle. Starovoitov suggested that changing the compiler to emit the tables differently might be the simplest solution. 

In fact, changing existing tools may be the only way to remain ""in distribution"" (within the area that LLMs have been trained on), he said. LLMs are not as good at handling tooling that isn't in their training sets; ensuring that BPF objects can be built and linked with existing tools is important to letting coding agents take advantage of the improvements. 

Amery Hung asked whether that extended to introducing new concepts, such as new kinds of BPF maps. Starovoitov thought that it did. When it was added, he had thought that BPF arenas would be the last map type needed. Since then the BPF subsystem has also added [ rhashtable-based maps](<https://lwn.net/Articles/1069677/>), but he doesn't envision the need for any more. 

The biggest change that will be required in the verifier is improved handling of loops, Starovoitov continued. There have been lots of struggles with BPF loops over the years; their handling has improved, but some loops are still too much for the verifier. Currently, the verifier will go through a loop multiple times until it proves one of three things: that the loop will exit after a number of iterations, that the loop returns to (a subset of) a previously analyzed state, or that the program will hit the maximum number of verified instructions and trigger an error. Starovoitov's proposed solution, called widening, would make it so that when verifying a loop takes more than a small number of iterations, the verifier state is generalized to consider larger numbers of loop-variable values at once ("widening" the state). The downside is that widened states are less precise, and potentially require more explicit run-time checks. This change could let the verifier handle all loops with a fixed maximum number of iterations, at the cost of requiring run-time checks for the loops that require widening. 

The early results of this approach are ""really great"", Starovoitov said, but there is a lot more work to do to make it usable. Still, he hopes to have something together in relatively short order. At that point, the scheduled session was coming to a close, but he had enough still to cover that the talk was extended to a second session the next day. 

#### Tracing and code review

The [ bpftrace](<https://bpftrace.org/>) and [DTrace](<https://dtrace.org/>) tools use domain-specific languages for configuring tracepoints; this makes them good for humans, but bad for LLMs, Starovoitov said. LLMs struggle with the syntax of niche languages. He thinks that the future of tracing looks like having documentation for coding agents that explains how to use standard tools, with explicit examples. That could help bpftrace and DTrace remain usable, but would be even better for tools like [ drgn](<https://github.com/osandov/drgn#drgn>) that use a standard language's syntax, just with extra capabilities. Ultimately, he wants to be able to tell a coding agent to ""find a performance issue on this kernel"" and have it use BPF and drgn to identify and debug the problem. 

One audience member thought that more examples in the documentation would be a good thing, but didn't think that using the kernel's tracing infrastructure with coding agents was viable unless one does not care about crashing the machine on which they are running. Starovoitov thought that this was ultimately solvable by encoding information about the kernel's functions and types in a format that can be checked automatically by existing tools, such as providing a "vmlinux.rs" set of Rust bindings. 

Starovoitov also wanted to discuss code reviews. The BPF subsystem is swamped with low-quality patches that look, on first impression, really good. At this point, he is ignoring the commit log on the assumption that it is written by LLMs and just looking directly at the patches themselves. But, in general, ""maintainers don't scale"". Starovoitov called on everyone who submitted patches to the BPF subsystem to also perform code reviews. 

There is no official number or scoring system, but contributing to BPF does require a fair and honest effort to review code. Cupertino Miranda asked whether he could invite people to come review code in GCC as well, since he is contributing to the kernel as part of his GCC BPF work. Starovoitov said that if he had a patch for GCC, he would certainly do some code review there as well, but ""I last touched GCC 20 years ago"". He did suggest that perhaps people could consider using LLMs to review their patches, in GCC and elsewhere. 

Steven Rostedt complained that not everyone has access to an infinite number of tokens, and that a dependency on cloud-hosted LLMs isn't sustainable. Starovoitov agreed that it was a dependency that needed to be managed, but didn't think that could be avoided: ""We are done with coding manually. It will all be agents in the future."" He then called it more of a way of life than a dependency. 

Rostedt objected that LLM companies don't have a proven business model, and so they're giving things away for free that they will not be able to afford to continue providing to kernel developers in the long term. Starovoitov did not think that LLMs were going to go away, even if prices rose in the future. Another audience member proposed that these companies should put together a fund for open-source developers, and let programmers pick which model to use, so that they would not be locked into one model that could disappear. Someone else pointed out that the [ Cloud Native Computing Foundation](<https://www.cncf.io/>) already does give out licenses for things like this to the maintainers of their projects. Starovoitov pointed out that open-source models also exist, and predicted that ""coding will be possible on a laptop, once the technology advances"". Rostedt agreed that things will become cheaper, but that didn't entirely ameliorate his concern. 

Borkmann said that the assembled BPF developers need to think about new people coming to the community. They should be able to review code, eventually, but there is a learning curve. How can they learn to do that when simple code review is done by LLMs? Starovoitov claimed that ""all of the newcomers, they're all skilled in agentic coding"". All of the patches that they are sending are written by coding agents, he said. It lowers the bar to kernel contribution. 

Rostedt didn't think this was a good thing — if ten times as many people are contributing, but the same quantity are sending good code, ""that's not a plus"". Starovoitov said that this is why more code review is needed. Sashiko has been helpful for that; he doesn't review patches until they've had an LLM comment. 

At that point, discussion broke up into a series of increasingly outré proposals for how to improve the review situation, such as asking LLMs to regenerate commit logs based on what is in the patches. Starovoitov gave one final announcement that BPF office hours (regular video meetings where one can discuss problems with the BPF maintainers) are now ad hoc, not regularly scheduled. People interested in attending one should send an email to the mailing list to find a time that works.
