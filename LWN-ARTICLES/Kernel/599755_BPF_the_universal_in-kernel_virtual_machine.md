---
title: "BPF: the universal in-kernel virtual machine"
url: https://lwn.net/Articles/599755/
date: "May 21, 2014"
category: BPF
author: "By Jonathan Corbet May 21, 2014"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 21, 2014

Much of the recent [discussion](<https://lwn.net/Articles/595565/>) regarding the Ktap dynamic tracing system was focused on the addition of a Lua interpreter and virtual machine to the kernel. Virtual machines seem like an inappropriate component to be running in kernel space. But, in truth, the kernel already contains more than one virtual machine. One of those, the BPF interpreter, has been growing in features and performance; it now looks to be taking on roles beyond its original purpose. In the process, it may result in a net reduction in interpreter code in the kernel. 

"BPF" originally stood for "Berkeley packet filter"; it got its start as a simple language for writing packet-filtering code for utilities like `tcpdump`. Support for BPF in Linux was added by Jay Schulist for the 2.5 development kernel; for most of the time since then, the BPF interpreter has been relatively static, seeing only a few performance tweaks and the addition of a few instructions for access to packet data. Things started to change in the 3.0 release, when Eric Dumazet [added a just-in-time compiler](<https://lwn.net/Articles/437981/>) to the BPF interpreter. In the 3.4 kernel, the "secure computing" (seccomp) facility was [enhanced](<https://lwn.net/Articles/475043/>) to support a user-supplied filter for system calls; that filter, too, is written in the BPF language. 

The 3.15 kernel sees another significant change in BPF. The language has now been split into two variants, "classic BPF" and "internal BPF". The latter expands the set of available registers from two to ten, adds a number of instructions that closely match real hardware instructions, implements 64-bit registers, makes it possible for BPF programs to call a (rigidly controlled) set of kernel functions, and more. Internal BPF is more readily compiled into fast machine code and makes it easier to hook BPF into other subsystems. 

For now, at least, internal BPF is entirely hidden from user space. The packet filtering and secure computing interfaces still accept programs in the classic BPF language; these programs are translated into internal BPF before their first execution. The idea seems to be that internal BPF is a kernel-specific implementation detail that might change over time, so chances are it will not be exposed to user space anytime soon. That said, the [documentation](<https://lwn.net/Articles/593476/#internals>) for internal BPF indicates that one of the goals of the project is to be easier for compilers like GCC and LLVM to generate. Given that any developer attempting to embed LLVM into the kernel has a rather small chance of success, that suggests that there may eventually be a way to load internal BPF directly from user space. 

This latter-day work has been done by Alexei Starovoitov, who looks set to continue improving BPF going forward. In 3.15, the just-in-time compiler only understands the classic BPF instruction set; in 3.16, it will be ported over to the internal format instead. Also, for the first time, the secure computing subsystem will be able to take advantage of the just-in-time compiler, speeding the execution of sandboxed programs considerably. 

Sometime after 3.16, use of BPF may be extended further beyond the networking subsystem. Alexei recently posted [a patch](<https://lwn.net/Articles/598545/>) that uses BPF for tracing filters. This is an interesting change that deletes almost as much code as it adds while improving performance considerably. 

The kernel's tracepoint mechanism allows a suitably privileged user to receive detailed tracing information every time execution hits a specific tracepoint in the kernel. As one might imagine, the amount of data that results from some tracepoints can be quite large. The NSA might be able to process such fire-hose-like streams at its new data center (once it's running), but many of the rest of us are likely to want to thin that stream down to something a bit more manageable. That is where the filtering mechanism comes in. 

Filters allow the association of boolean expression with any given tracepoint; the tracepoint only fires if the expression evaluates to true at execution time. An example given in [Documentation/trace/events.txt](<https://lwn.net/Articles/599766/>) reads like this: 

```
# cd /sys/kernel/debug/tracing/events/signal/signal_generate
        # echo "((sig >= 10 && sig < 15) || sig == 17) && comm != bash" > filter
```

With this filter in place, the `signal_generate` tracepoint will only fire if the specific signal being generated is within the given range and the process generating the signal is not running `bash`. 

Within the tracing subsystem, an expression like the above is parsed and represented as a simple tree with each internal node representing one of the operators. Every time that the tracepoint is encountered, that tree will be walked to evaluate each operation with the specific data values present at the time; should the result be true at the top of the tree, the tracepoint fires and the relevant information is emitted. In other words, the tracing subsystem contains a small parser and interpreter of its own, used for this one specific purpose. 

Alexei's patch leaves the parser in place, but removes the interpreter. Instead, the predicate tree produced by the parser is translated into an internal BPF program, then discarded. The BPF is translated to machine code by the just-in-time compiler; the result is then run whenever the tracepoint is encountered. From the benchmarks posted by Alexei with the patch, the result is worth the effort: the execution time for most filters is reduced by a factor of approximately twenty — and sometimes quite a bit more. Given that the overhead of tracing can often hide the very problems that tracing is being used to find, a huge reduction in that overhead can only be welcome. 

The patch set was indeed welcomed, but it is unlikely to find its way into the 3.16 kernel. It currently depends on the other 3.16 changes, which are merged into the net-next tree; that tree is not normally used as a dependency for changes elsewhere in the kernel. As a result, merging Alexei's changes into the tracing tree creates compilation failures — an unwelcome result. 

The root problem here is that the BPF code, showing its origins, is buried deeply within the networking subsystem. But usage of BPF is no longer limited to networking code; it is being employed in core kernel subsystems like secure computing and tracing as well. So the time has come for BPF to move into a more central location where it can be maintained independently of the networking code. This change is likely to involve more than just a simple file move; there is still a lot of networking-specific code in the BPF interpreter that probably needs to be factored out. It will be a bit of work, but that is normal for a subsystem that is being evolved into a more generally useful facility. 

Until that work is done, BPF-related changes to non-networking code are going to be difficult to merge. So _that_ is the logical next step if BPF is to become the primary virtual machine for interpreted code loaded into the kernel. It makes sense to have only one such machine that, presumably, is well debugged and maintained. There are no other credible contenders for that role, so BPF is almost certainly it, once it has been repackaged as a utility for the whole kernel to use. After that happens, it will be interesting to see what other users for BPF come out of the woodwork.
