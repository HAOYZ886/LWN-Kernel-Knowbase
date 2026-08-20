---
title: Scrutinizing bugs found by syzbot
url: https://lwn.net/Articles/872649/
date: "October 13, 2021"
category: "Development tools-Fuzz testing"
author: "By Jake Edge October 13, 2021 LSSNA"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
October 13, 2021

* * *

[LSSNA](<https://lwn.net/Archives/ConferenceByYear/#2021-Linux_Security_Summit_North_America>)

The [syzbot kernel-fuzzing system](<https://github.com/google/syzkaller/blob/master/docs/syzbot.md>) finds an enormous number of bugs, but, since many of them may seem to be of a relatively low severity, they have a lower priority when contending for the attention of developers. A talk at the recent [Linux Security Summit North America](<https://events.linuxfoundation.org/linux-security-summit-north-america/>) reported on some research that dug further into the bugs that syzbot has found; the results are rather worrisome. Rather than a pile of difficult- or impossible-to-exploit bugs, there are numerous, more serious problems lurking within. 

#### SyzScope

The first speaker on day one of the summit was Xiaochen Zou, a PhD student at the University of California, Riverside, who described a tool that he and his colleagues have been running on bugs that syzbot—which uses the [syzkaller coverage-guided fuzzer](<https://github.com/google/syzkaller>)—has reported in the Linux kernel. Over the last four years, syzbot has reported around 4000 bugs, of which 3000 have been fixed. There are eight categories that these bugs mainly fall into, but only some of those categories represent security bugs, he said. These are classic, severe kernel security holes, such as use after free, double free, and out-of-bounds writes; they are mostly found by the [Kernel Address Sanitizer](<https://www.kernel.org/doc/html/v4.14/dev-tools/kasan.html>) (KASAN) as part of syzkaller runs. 

[ ![\[Xiaochen Zou\]](https://static.lwn.net/images/2021/lssna-zou-sm.jpg) ](<https://lwn.net/Articles/872881/>)

There are several other classes of bugs that need fixing, but they tend to get less attention; these include kernel or sanitizer assertions and general protection faults (e.g. invalid pointer dereference). In addition, bugs that allow memory to be read can provide information leaks, such as kernel addresses, but those are generally not needed for exploits to compromise the kernel, Zou said. The researchers classified these as low-risk bugs. Bugs that are in lower-risk categories take longer to fix on average, and longer to backport to various older kernels. The sheer number of bugs that syzbot reports overwhelms the ability of the kernel community to fix them everywhere they occur. 

If the bugs that are slower to be fixed are truly of low risk, that's less of a problem, but the research found that many of these bugs are not actually low risk. Beyond that, the syzbot reports do not always tell the full story of the bugs' impacts. Even worse still, there are ways to automatically find higher-risk impacts from some of these low-risk bugs. [SyzScope](<https://github.com/seclab-ucr/SyzScope>) is a tool that sets out to reveal high-risk impacts of bugs that are reported in a low-risk category. SyzScope is not meant to automatically create full exploits because there are already other research projects that are solving that part of the problem. In particular, [FUZE](<https://www.usenix.org/conference/usenixsecurity18/presentation/wu-wei>) focuses on doing so for use-after-free vulnerabilities, and [KOOBE](<https://www.usenix.org/conference/usenixsecurity20/presentation/chen-weiteng>) can be used for out-of-bounds-write vulnerabilities. 

There were several insights that the researchers had which led to the development of SyzScope, Zou said. Syzbot only shows the first problem it finds in an execution path that it tests; it normally stops and reports the bug. That makes sense, because syzbot is looking for bugs, not necessarily security-related bugs. But it may ignore further, more risky impacts downstream from the bug it is reporting, which may downplay the risks of the bug, thus possibly delay it from being fixed promptly. 

Even if you allow fuzzing to continue, though, there are several types of impacts that it will not be able to find, he said. Control-flow hijacking and the writing of values or addresses (either arbitrarily or constrained in some way) are not detected by the sanitizers, kernel assertions, or other mechanisms used by syzkaller. 

He gave an example of an out-of-bounds read that KASAN detected and syzbot reported; he showed that it could be turned into a write of null to an arbitrary address. But, because it relies on specific values in out-of-bounds memory, fuzzing will not be able to find it; only if the out-of-bounds memory is specially prepared will it be useful for this purpose. 

[Heap spraying](<https://en.wikipedia.org/wiki/Heap_spraying>) is a technique that an attacker could use to arrange memory in that fashion. Beyond just writing null, though, even further downstream from the syzbot-related bug is a function call made to an address that comes from the out-of-bounds memory, which can be controlled by an attacker. That leads to control-flow hijacking and a full compromise of the kernel. 

But those further impacts were found by human experts, not by syzkaller or some other tool. The researchers wanted a way to discover those types of problems automatically. Fuzzers cannot control the contents of the objects that are used after they are freed or the contents of out-of-bounds memory like an attacker can. 

#### Symbolic execution

There is a technique that can do that kind of work, however: [symbolic execution](<https://en.wikipedia.org/wiki/Symbolic_execution>). By using symbolic values rather than concrete values during program analysis, symbolic execution can effectively simulate the results of heap spraying. It can show what values need to be stored, and where to place them, to cause the function to be called at a location of the attacker's choosing, for example. 

There are two modes of operation for SyzScope, both of which search for high-risk bugs lurking behind low-risk reports. In one mode, it uses static analysis and symbolic execution on open bugs reported by syzbot. In the other, it looks at bugs that have been fixed, using fuzzing, static analysis, and symbolic execution. The latter incorporates fuzzing to find downstream effects of the original bug; it uses the bug-fix patch to validate that new bugs found are consequences of the original. 

In the interests of time, Zou said he would skip talking about the static-analysis piece; it is optional and simply assists the symbolic-execution step. It is, in effect, an optimization. 

A restricted form of fuzzing is used to try to find additional buggy contexts that share the same root cause as the original bug; those are then used by the symbolic-execution step to see if they are actually high-risk bugs. Unlike syzbot, this fuzzing, which also uses syzkaller, does not stop when it finds a bug. Instead, it continues to run to see what else it can find. The fuzzer starts with the proof-of-concept code that syzkaller produced to demonstrate the original problem and then uses a "conservative mutating strategy" to try to find use-after-free bugs, out-of-bounds writes, and the like. It uses the bug-fix patch to determine which new contexts are related to the original fixed bug; if those new contexts do not trigger a complaint from the patched kernel, they correspond to high-risk bugs hiding underneath the report that seemed to be of a low-risk bug type. 

The fuzzer augments syzkaller's code-coverage feedback with impact feedback. It tries to find bugs that have a higher impact in order to find high-risk bugs. The code-coverage feedback sometimes leads the fuzzer into finding entirely new bugs, which is not the focus of their efforts. 

The symbolic-execution phase uses QEMU; it sets a breakpoint at the place where KASAN reported a problem and triggers the bug. It then launches "angr", which is the symbolic-execution engine; angr finds the out-of-bounds memory addresses from the KASAN report and "symbolizes" them. The register values in QEMU are transferred to angr, which can also retrieve memory contents dynamically from QEMU. 

The engine looks for certain kinds of assembly language instructions, that operate on values from the range of out-of-bounds memory reported by KASAN. This allows it to detect ways that various types of flaws can be exploited. For example, if the value passed to `kfree()` comes from within the controllable memory, it is flagged as an invalid free; similarly, a call to an address that comes from out-of-bounds memory is a way to produce a control-flow-hijacking exploit. Five different types of high-risk impacts are detected in this way. 

#### Results

The researchers ran an experiment using nearly 1200 lower-risk bugs that syzbot reported. They ran each with three hours of fuzzing and four hours of symbolic execution. Overall, the experiment was able to find high-risk impacts for 147 low-risk bugs. A lot of those low-risk bugs had multiple other impacts, many of them from high-risk categories, associated with them. For example, 51 control-flow-hijacking impacts were found, both from fixed and open bugs that syzbot had reported in lower-risk categories. Around 3200 separate impacts beyond what syzbot reported were found. 

Just the fuzzing discovered around half of the high-risk bugs, with roughly 5% of the total impacts found. As noted earlier, fuzzing can only find impacts that do not require control of the out-of-bounds memory contents. For bugs that had already been fixed, thus could use the fuzzing stage, there were an average of nearly 28 further impacts per kernel bug tested; an average of roughly 17 additional impacts were found from the open bugs tested. 

Symbolic execution detected the other half of the high-risk bugs found in the experiment, including all 34 that were found from bugs that were reported by syzbot but still unfixed in the kernel. It also found the vast majority (95%) of the impacts, including all of super-high-risk impacts, like control flow hijacking and arbitrary address writes. 

The researchers submitted 32 high-risk bugs to the CVE maintainers, eight of which were assigned CVE numbers (which can be seen in slide 28 of Zou's [slides](<https://static.sched.com/hosted_files/lssna2021/55/SyzScope%20in%20Linux%20Security%20Summit.pdf>)). Their paper will be presented at the [31st USENIX Security Symposium](<https://www.usenix.org/conference/usenixsecurity22>) in August 2022; it should be available soon, Zou said. 

The voluminous output from syzbot and the inability of kernel developers to keep up with the onslaught has been a frequent cause for concern over the years; these findings can only heighten those fears. It would not be surprising to find out that black hats and governmental agencies are using similar techniques to turn seemingly semi-innocuous bugs into ways to fully compromise the kernel—in fact, it would be surprising if they were _not_. To a certain extent, these findings show that prioritizing bugs based on the impact reported by syzbot probably frequently downplays the danger, but they do tend to validate the "all bugs are security bugs" attitude that prevails within parts of the kernel-development community.
