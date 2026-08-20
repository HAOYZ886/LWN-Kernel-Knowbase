---
title: "BPF: what's good, what's coming, and what's needed"
url: https://lwn.net/Articles/787856/
date: "May 9, 2019"
category: BPF
author: "By Jonathan Corbet May 9, 2019 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 9, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The 2019 Linux Storage, Filesystem, and Memory-Management Summit differed somewhat from its predecessors in that it contained a fourth track dedicated to the BPF virtual machine. LWN was unable to attend most of those sessions, but a couple of BPF-related talks were a part of the broader program. Among those was a plenary talk by Dave Miller, described as "a wholistic view" of why BPF is successful, its current state, and where things are going. 

Years ago, Miller began, Alexei Starovoitov showed up at a netfilter conference promoting his ideas for extending BPF. He described how it could be used to efficiently implement various types of switching fabric — _any_ type, in fact. Miller said that he didn't understand the power of this idea until quite a bit later. 

#### What's good now

BPF, he said, is well defined and useful, solving real problems and allowing users to do what they want with their systems. It is increasingly possible to look at what is going on inside the kernel or to modify its behavior without having to change the source or reboot the system. BPF provides strict limits on what programs can do, running them inside the kernel but in a sandboxed mode. BPF programs operate on a specific [![\[Dave Miller\]](https://static.lwn.net/images/conf/2019/lsfmm/DaveMiller-sm.jpg)](<https://lwn.net/Articles/787859/>) object (called the "context"), and there are many places to attach them. They execute in finite time; something that is assured because they are not currently allowed to contain loops (though that will change to some extent eventually). BPF maps provide data structures for programs and can be used to share access to data. 

The BPF verifier, Miller said, is "the only thing between us and extreme peril". It is the last line of defense preventing dangerous code from getting into the system. It is so good, he said, that it often frustrates BPF authors, who have to massage their programs to get them to a point where the verifier will accept them. 

The real value of BPF lies in the fact that "we are all arrogant". System designers tend to think that they know what their users want to do, so they make boxes to enable that one thing. But users don't want to be in a box; those users are a constant source of new ideas, and developers often don't know what they want. BPF allows users to escape the box created by system designers, who may be sad about that, but they'll get over it. 

BPF has been growing slowly, by word of mouth, Miller said, because there is no "advertising machine" for this technology. Users are still learning about it. The good news is that, once technical people get into a new technology, they tend to spread it around. That has happened with BPF, to the point that people are now writing books about it. 

#### What's improving

BPF is mostly useful now for solving simple problems, Miller said, but it is rapidly gaining the ability to deal with "real programs". One step in that direction is increasing the size limit for BPF programs from its current value of 4096 instructions to 1 million. The prohibition on loops forces developers to unroll loops in their programs, which is unfortunate; the in-development [support for bounded loops](<https://lwn.net/Articles/773605/>) will fix that problem someday. 

BPF programs can perform tail calls now; they function a lot like continuations. Tail calls are a great way to build an execution pipeline, where each step in the pipeline performs a tail call to the next. But now BPF is able to support real function calls as well, but a given program still cannot use both due to limitations in the verifier. 

One area where BPF has seen some improvement is introspection; it can be hard for developers to understand why their program is not doing what they want. Indeed, in current kernels, it's hard even to determine which BPF programs have been loaded into the system, or to verify that a loaded program is the one that is wanted. The `bpftool` utility is improving in its introspection support, as is the BTF format for describing data types, which will help to [increase the portability](<https://lwn.net/Articles/773198/>) of BPF programs. BTF turns out to be good for annotating BPF programs and how they work. The `perf` utility is also gaining the ability to drill down into BPF programs. Users cannot complain, Miller said, that visibility into BPF is not being provided. 

#### What's needed

There is no shortage of opportunities for improvement still, he said. For example, BPF does not currently support code reuse all that well; there are a lot of people out there writing their own Ethernet header parsers. There are systems with thousands of redundant BPF programs loaded into them; that is not the way to do software development. Support for function calls will help, but BPF needs libraries, and it will need access control for those libraries once they can be loaded. BTF will help, since it will make it easy to see which libraries are available in a given system. 

BPF development is still harder in general than it should be; Miller would like to see the development of a "type and go" environment that makes writing and loading a BPF program as easy as on the Arduino. Unskilled people should be able to get stuff done; that is part of the goal of wresting control away from arrogant system developers. 

BPF programs should have "trivial debuggability", he continued. It should be possible to single-step through programs and examine context data. He would like the ability to record a program's execution or state so that it could be stepped through outside of the kernel. Perhaps even live, in-kernel single-stepping could be supported in development environments. The most important thing for the near future, though, is the ability to snapshot the current state of a BPF program. 

Finally, he said, BPF needs better access control. Almost all BPF functionality is root-only now, but things will not be that way forever. Much more granular control to BPF functionality is required — or we could always control access to BPF with a BPF program, he said. A file like `/dev/bpf` could be used for access control, but that's still pretty coarse; perhaps what is needed is a hierarchy of files describing the different program types and their access permissions. BPF also needs better memory accounting, since maps can get quite large. 

At this point, Miller concluded his talk and accepted questions. Matthew Wilcox started things off by saying that he will not be impressed by BPF until it becomes possible to play [Zork](<https://en.wikipedia.org/wiki/Zork>) in the kernel. The original Zork was less than 1 million instructions, Wilcox said, so that should be possible. 

#### ABI compatibility

The first actual question was about whether there are any inherent limits on what BPF will eventually be able to do. Early on, Miller answered, it was used for tasks like packet analysis, and current usage still reflects that. BPF will not be usable to implement a proprietary TCP stack in the kernel, for example; that is not a goal. Among other things, there are no timers available to BPF programs and no plans to add them. 

Some people do try to push the limits, Miller said. Steve Hemminger tried to convert a packet scheduler to BPF, for example, but eventually ran into the timer issue. Somebody else, though, managed to create a complete implementation of OpenVSwitch, but that sort of project really misses the point of BPF. The real value is not in doing everything, but in being able to do exactly what you need and no more. 

Ted Ts'o said that he did not expect to see device drivers in BPF, but Miller responded that those already exist. He was referring to the ability to [perform infrared protocol decoding](<https://lwn.net/Articles/759188/>) in a BPF program. That eliminates the need to support hundreds of infrared devices in a kernel driver and allows support for new devices to be easily added to older kernels. Ts'o conceded that point, but said that it was unlikely that there would be an NVIDIA GPU driver written in BPF anytime soon. 

Another attendee asked about ABI compatibility; will the kernel have to support existing BPF programs forever? Miller responded that BPF exists in an "ambiguous plane" between kernel ABI and the "wild west" of the kernel's internals. Tools like BTF will help to make things more compatible over time. Meanwhile, the BPF developers have taken liberties to break things in the early stages; the community is still learning how all of this stuff should work. But that should happen less often over time. That said, he doesn't think it will ever be possible to write a BPF program and expect that it will work on every future kernel. 

The discussion turned to the [`powertop` episode](<https://lwn.net/Articles/442113/>), where a change to a tracepoint broke the `powertop` utility and had to be reverted. As a result of that, some maintainers still refuse to allow the addition of tracepoints in their subsystems. The problem is that `powertop` was useful, so users complained when it broke. BPF programs, too, will be useful, and are likely to suffer from the same problems. Brendan Gregg may have [said](<https://lwn.net/Articles/787131/>) earlier in the week that occasional breakage was OK, but someday some user will complain and Linus Torvalds will revert a BPF-visible change. Miller responded that, whenever a new facility like this is added, there is always a period in which things break. We'll never get away from that, but it will get a lot better. 

Ts'o worried about how bad the ABI pain would be; some BPF interfaces will not be changeable, he said. At least, it will not be possible to change them without a ten-year deprecation period while old programs are fixed. Miller said that, with BPF, users are often happy when things break, because it usually indicates that new information is available for them to work with. 

Gregg said that, in the absence of tracepoints, current BPF tools are using a lot of kprobes. There are a lot of kernel-version checks that go with them, but they still break with every kernel release. If the kernel moves to tracepoints that only break once every five years, that will be fantastic. Ts'o wondered whether the breakage of a kprobe-based tool that is seen as being as useful as `powertop` would cause Torvalds to revert a change. He does not know the answer to that. 

#### Security

Dave Hansen asked about security and side channels; BPF was one way in which the Spectre vulnerabilities could be exploited early on. These issues have been mitigated one at a time as they are found, but has any thought been given to broader mitigations? Miller acknowledged that programs can be written to exploit speculative execution vulnerabilities; the verifier can often detect and block such attempts. On the other hand, BPF can also improve security. He mentioned an episode where a bug in a custom hash computation could be exploited to crash the kernel; it was possible to move the computation to BPF and block exploits until the kernel was fixed. Hansen continued, saying that the kernel-hardening efforts are trying to address problems proactively; work in the BPF area, he said, is more reactive. Miller conceded that point, but said that, hopefully, the kernel is becoming sufficiently hardened that it will no longer be necessary to worry about these issues all the time. 

The final question came from Ts'o, who wondered about how BPF will interact with Linux security modules. With the advent of [stackable security modules](<https://lwn.net/Articles/635771/>), it should be possible to implement more flexible access control for BPF programs. He also suggested that perhaps some verifier policies should include interaction with the security-module subsystem. 

Miller answered that the verifier has a set of operations specific to each program type; it should be possible to add a security-module hook there somehow. He also observed, with amusement, that SELinux is using classic BPF now for a few things. It would be great, he said, to use BPF to create new security policies; it could be the "universal security policy engine". That would allow for the immediate addition of new policies without the need to wait for the next kernel release.
