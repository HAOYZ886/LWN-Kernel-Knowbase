---
title: A strange BPF error message
url: https://lwn.net/Articles/1016105/
date: "April 4, 2025"
category: "BPF-Verifier"
author: "By Daroc Alden April 4, 2025 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Daroc Alden**  
April 4, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Yonghong Song brought a story about tracking down the cause of a strange verifier error message to the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit. He then presented some possible ways to improve Clang's user experience for anyone running into the same class of error in the future. Toward the end of his allotted time, he also discussed the problems with optimizations that change the signature of functions — a problem that José Marchesi had also brought up in [the previous session](<https://lwn.net/Articles/1015747/>). 

#### An unhelpful error

Song started by presenting an example taken from a real program. The example was a bit dense, but the problem basically comes down to this code: 

```
bool icmp6_ndisc_validate() {
            __u8 nexthdr;
            // ...
            int offset = ipv6_hdrlen_offset(&nexthdr);
            // ...
        }
    
        static __always_inline int ipv6_hdrlen_offset(__u8 *nexthdr) {
            __u8 nh = *nexthdr;
            // ...
            switch (nh) {
            case NEXTHDR_NONE:
                return DROP_INVALID_EXTHDR;
            // ...
            }
        }
```

The code features an uninitialized variable (nexthdr) that was passed by reference into another function. This is not invalid in C, because the other function might initialize the variable by writing to it, so Clang doesn't issue a warning. In this case, though, `ipv6_hdrlen_offset()` does not initialize it, and instead reads from it in order to decide which branch of a switch statement to take. Clang doesn't warn in that function either, because it assumes the function argument points to initialized memory. 

At that point, the code is passed to the optimizer, and everything goes wrong. The optimizer inlines one function into the other function, notices that the program is reading from an uninitialized variable (which would be undefined behavior, which it assumes cannot happen), and decides that this code must be unreachable. It turns the entire tail of the function into a single `unreachable` instruction in the LLVM IR, and then hands that off to the BPF code-generation backend. That backend ignores the `unreachable` instruction, but since the function's original return has been subsumed into it, the code generator ends the function without emitting a return instruction. That leads in turn to this somewhat confusing error from the BPF verifier: 

```
last insn is not an exit or jmp
```

While this error makes sense with an understanding of the sequence of events that led to it, at first Song found it a good deal more puzzling. It's not intuitive that an uninitialized variable would cause this error message, he said. He actually ran into this same problem helping someone with another program — so this isn't an isolated incident. People are seeing this message and being justifiably confused. 

Marchesi and David Faust said that GCC does pretty much the same thing, and therefore has pretty much the same problem. One audience member asked why LLVM was generating an `unreachable` instruction instead of inferring that the value of the variable was `undef` (LLVM's representation of a value which could be anything). Song answered that LLVM's `undef` has ""a lot of interesting semantics"" that made it not always the right fit. 

[ ![\[Yonghong Song\]](https://static.lwn.net/images/2025/yonghong-song-small.png) ](<https://lwn.net/Articles/1016238/>)

There have been a few attempts to avoid this kind of error, Song said. One option is to use `-ftrivial-auto-var-init=zero` to make the compiler initialize all variables with zero, where possible. This sort of works, in the sense that the generated program is no longer rejected, but it may hide a real bug. It's also a performance problem for some [ express data path](<https://en.wikipedia.org/wiki/Express_Data_Path>) (XDP) programs that may need to initialize lots of IP headers. 

Another approach that he tried was to have the BPF backend recognize the `unreachable` instruction and emit an error at that point. This is better, but it's not an airtight defense, because there's no guarantee that the optimizer won't do something else in the future that results in different code being generated. For example, it could have just chosen to assume that the value of the variable matched whichever switch statement it found most convenient. 

If the presence of `unreachable` could be relied on, the BPF backend could emit a useful error message when it sees it. So the approach Song is currently pursuing is to try and make it so that the optimizer will not use transformations that can eliminate `unreachable` instructions when compiling for BPF. He also has [a pull request](<https://github.com/llvm/llvm-project/pull/131020>) for LLVM open that tries to generate `unreachable` in more cases, although it looks unlikely to be accepted in its current form. 

One attendee suggested using LLVM's `poison` value, which is subtly different from `undef` (as [a presentation](<https://www.youtube.com/watch?v=ZMaZH3YYJqY>) from the 2020 LLVM developer's meeting explains). Song agreed that it was possible in theory, but it wasn't likely to be accepted by the LLVM maintainers for various reasons. 

Marchesi wondered whether this same kind of behavior could manifest in other verifier errors, or whether it was always the same message. Song answered that he had only observed this specific error in testing, but that in general there was no reason to assume that other verifier errors were impossible. Eduard Zingerman said that he had actually seen some sched_ext code that did not result in the "last insn is not an exit or jmp" message, but had caused a verification failure in a different place in the program. Marchesi suggested that this specific case could be caught by examining the program's control-flow graph at compile time. Song said that was not possible, because LLVM's BPF backend doesn't have access to the control-flow graph. Marchesi asserted that this was a problem with LLVM's design, and that the backend needs access to the program's control-flow graph for several reasons. 

As a partial solution that would at least deliver better error messages, Song proposed having the BPF backend generate a call to the non-existent `bpf_unreachable()` kernel function when it sees a `unreachable` instruction. This would still result in a verifier failure on existing kernels, but hopefully one that is more specific and therefore easily searched for. Future kernels could recognize calls to `bpf_unreachable()` and supply a nicer failure message. Specifically, he proposed: 

```
last isns marked as unreachable, maybe due to uninitialized variable?
```

Some other alternatives he considered included adding an `unreachable` instruction to the BPF virtual machine, adding a `bpf_unreachable()` kernel function, or actually making the Clang frontend detect all uninitialized variable usage across functions. The first two are not really necessary, he said. Someone working at Google actually had a patch that implemented the latter option, but it never got merged. At the time, the project didn't consider it a priority because people normally use a sanitizer to detect problems with uninitialized variables. Unfortunately, that's not really an option for BPF programs. 

Faust commented that this sounded like another use case where it would be helpful to have the rules of the verifier extracted out of the kernel so they could be run elsewhere. If that were done, the compiler could check the binary itself, and then use its context on the program to produce a more helpful error message. 

#### Signature changes

With the time remaining in the session, Song turned to another topic: how optimization can change the signatures of functions, and how to represent that in BPF's debugging information format, BTF. According to an analysis of the DWARF debug information of a recent kernel, there are 64,129 functions in the kernel. Of those, 635 have arguments changed, 306 have the return value removed, and 18 have both. 

The DWARF debug format does actually have a way to represent that information, in the form of the `DW_AT_calling_convention` tag, but it's not specific enough — it only tells the user that something changed, not what changed. Song then briefly described two proposed new ways of representing the original signature of an optimized function in BTF. Unfortunately, the group didn't have much time to dig into the the topic before it was time for the next session.
