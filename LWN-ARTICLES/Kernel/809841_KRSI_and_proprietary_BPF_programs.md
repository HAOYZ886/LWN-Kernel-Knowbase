---
title: KRSI and proprietary BPF programs
url: https://lwn.net/Articles/809841/
date: "January 17, 2020"
category: "BPF-Security; Security-Security modules"
author: "By Jonathan Corbet January 17, 2020"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 17, 2020

The "kernel runtime security instrumentation" (or KRSI) patch set enables the attachment of BPF programs to every security hook in the kernel; LWN [covered this work](<https://lwn.net/Articles/808048/>) in December. That article focused on ABI issues, but it deferred another potential problem to [our 2020 predictions](<https://lwn.net/Articles/808260/>): the possibility that vendors could start shipping proprietary BPF programs for use with frameworks like KRSI. Other developers did pick up on the possibility that KRSI could be abused this way, though, leading to a discussion on whether KRSI should continue to allow the loading of BPF programs that do not carry a GPL-compatible license. 

It may be surprising to some that the kernel, while allowing BPF programs to declare their license, is entirely happy to load programs that have a proprietary license. This behavior, though, is consistent with how the kernel handles loadable modules: any module can be loaded, but modules without a GPL-compatible license will not have access to many kernel symbols (any that are exported with `EXPORT_SYMBOL_GPL()`). BPF programs interact with the kernel through special "helper functions", each of which must be explicitly exported; these, too, can have a "GPL only" marking on them. In current kernels, about 25% of the defined helpers are restricted to GPL-licensed code. 

The issue of licensing for KRSI programs was first [raised](<https://lwn.net/ml/linux-kernel/e90e03e3-b92f-6e1a-132f-1b648d9d2139@tycho.nsa.gov/>) by SELinux maintainer Stephen Smalley, who noted: 

And a traditional security module would necessarily fall under GPL; is the eBPF code required to be likewise? If not, KRSI is a gateway for proprietary LSMs. 

After a bit of discussion, KRSI developer KP Singh [suggested](<https://lwn.net/ml/linux-kernel/20200110152758.GA260168@google.com/>) that he could add an explicit check for GPL-compatible licensing to KRSI. That might have resolved the issue right there, but Alexei Starovoitov [objected to this change](<https://lwn.net/ml/linux-kernel/20200110175304.f3j4mtach4mccqtg@ast-mbp.dhcp.thefacebook.com/>), saying that BPF programs should continue to be treated like loadable modules, and that the solution was simply to ensure that all of the helper functions needed to get real work done were marked GPL-only. 

There is a bit of a problem with this reasoning, though, related to the specific role that BPF programs fill in KRSI. While the KRSI developers envision using BPF programs to do a variety of system-wide monitoring tasks, these programs also function as security hooks in their own right. They have full read access to the kernel data structures passed into those hooks and, in the end, they are charged with returning a Boolean value carrying their verdict as to whether a given operation should be allowed to proceed or not. There is a fairly wide range of security policies that could be implemented in BPF without calling any helper functions at all. Smalley [pointed this out](<https://lwn.net/ml/linux-kernel/554ab109-0c23-aa82-779f-732d10f53d9c@tycho.nsa.gov/>) and added: 

It seems like the question is whether the kernel developers are ok with exposing the entire LSM hook interface and all the associated data structures to non-GPLd code, irrespective of what helpers it may or may not use. 

Starovoitov [was unmoved](<https://lwn.net/ml/linux-kernel/20200115024830.4ogd3mi5jy5hwr2v@ast-mbp.dhcp.thefacebook.com/>) by this concern, though, and argued that it should be possible to load simple non-GPL programs: 

Primitive KRSI progs like 

```
int bpf-prog(void*) { return REJECT; }
```

would be able to selectively disable a syscall with an overhead acceptable in production systems (unlike seccomp). I want this use case to be available to people. It's a bait, because to do real progs people would need to GPL them. Key helpers bpf_perf_event_output, bpf_ktime_get_ns, bpf_trace_printk are all GPL-ed. It may look that most networking helpers are not-GPL, but real life is different. To debug programs bpf_trace_printk() is necessary. To have communication with user space bpf_perf_event_output() is necessary. To measure anything or implement timestamps bpf_ktime_get_ns() is necessary. 

So today all meaningful bpf programs are GPL. Those that are not GPL probably exist, but they're toy programs. Hence I have zero concerns about GPL bypass coming from tracing, networking, and, in the future, KRSI progs too. 

Smalley [was not convinced](<https://lwn.net/ml/linux-kernel/38a82df5-7610-efe1-d6cd-76f6f68c6110@tycho.nsa.gov/>), saying: ""I would anticipate developers of out-of-tree LSMs latching onto this bpf LSM and using it to avoid GPL"". Greg Kroah-Hartman [dropped into the conversation](<https://lwn.net/ml/linux-kernel/20200115140953.GB3627564@kroah.com/>) to support Smalley: ""We have enough problem with companies trying to do that as-is, let's not give them any other ways to abuse our license"". Starovoitov [threw in the towel](<https://lwn.net/ml/linux-kernel/20200115222314.wiqamvax7vckgfv7@ast-mbp.dhcp.thefacebook.com/>) at this point, consoling himself that developers can always ""undo it later when this early concerns prove to be overblown"". 

As a result of this discussion, the [second posting of KRSI](<https://lwn.net/ml/linux-kernel/20200115171333.28811-1-kpsingh@chromium.org/>) included the license check, though this was not noted in the changelog. This version also eliminated the securityfs-based API in favor of using the `bpftool` utility for configuration and introspection of security policies. 

While there have been some minor comments on this version of the patch set, it may well be that the biggest roadblock to its merging has now been overcome. Other concerns, including the possibility of ABI-related problems in the future and the implications of allowing the attachment of BPF programs — with access to internal kernel data structures and veto power — to almost every operation user space can attempt, do not appear to worry many people. As a result, KRSI could conceivably be headed toward the mainline in the near future.
