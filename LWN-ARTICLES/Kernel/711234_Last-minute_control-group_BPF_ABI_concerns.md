---
title: "Last-minute control-group BPF ABI concerns"
url: https://lwn.net/Articles/711234/
date: "January 11, 2017"
category: BPF; Control groups
author: "By Jonathan Corbet January 11, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 11, 2017

One of the features pulled into the mainline during the 4.10 merge window is the ability to attach a BPF program to a control group; that program can then filter packets received or transmitted by processes within the control group. The feature itself is relatively uncontroversial (though [some would prefer](<https://lwn.net/Articles/698073/>) a different implementation). Until recently, the feature's interface and semantics were also uncontroversial — or at least not closely examined. Since the feature was merged, however, some concerns have been raised. The development community will have to decide whether changes need to be made, or the feature temporarily disabled, before the 4.10 release sets the interface in stone. 

The conversation was [started by](<https://lwn.net/Articles/711235/>) Andy Lutomirski, who played with the new capability for a while and found a few things that worried him. The first of these is that the `bpf()` system call is used to attach the program to the control group. This is, he thinks, fundamentally a control-group operation, not a BPF operation, so it should be handled through the control-group interface. If, in the future, somebody adds the ability to impose other types of controls — controls that don't involve BPF programs — then the use of `bpf()` would make no sense. And, in any case, he said, `bpf()` is an increasingly unwieldy multiplexer system call. 

This objection didn't get far; there does not seem to be a large contingent of developers interested in adding other packet-filtering mechanisms to control groups. BPF developer Alexei Starovoitov [dismissed](<https://lwn.net/Articles/711238/>) the idea, suggesting that any other mechanism could be just as easily implemented in BPF. Networking maintainer David Miller [agreed](<https://lwn.net/Articles/711239/>) with Starovoitov on this issue, so it seems that little is likely to change on this point. 

The next issue runs a little deeper. Control groups are hierarchical in nature and, with version 2 of the control-group interface, all controllers are expected to behave in a fully hierarchical manner. The BPF filter mechanism is not a proper controller (a bit of an interface oddity in its own right), but its behavior in control-group hierarchies is still of interest. Controller policies are normally composed as one moves down the hierarchy. For example, if a control group is configured with the CPU controller to have 10% of the available CPU time, then a sub-group of that group is configured to get 50%, it will end up with 50% of the 10% the parent group has, or 5% in absolute terms. 

If a process is running in a two-level control group hierarchy, where both levels have filter programs attached, one might think that _both_ filters would be run — that the restrictions imposed by those filters would be additive. But that is not what happens; instead, only the filter program at the lowest level is run, while those at higher levels are ignored. The upper level filter might prohibit certain kinds of traffic, but the mere existence of a lower-level filter overrides that prohibition. In a setting where one administrator is setting filters at all levels, these semantics might not be a problem. But if one wants to set up a system with containers and user namespaces, where containers can add filter programs of their own, this behavior would allow the system-level policy to be circumvented. 

Starovoitov [acknowledged](<https://lwn.net/Articles/711242/>) that, at a minimum, there might be a use case for composing all the filters in a given hierarchy. But he also asserted that ""the current semantics is fine for what it's designed for"" and said that different behavior can be implemented in the future. The problem with that approach is that changing the semantics would be a significant ABI change that could easily break systems that were designed around the 4.10 semantics; such a change would not be allowed. In the absence of a plan for how the new semantics could be added in a compatible way, it has to be assumed that, if 4.10 is released with the current behavior, nobody will be able to change it going forward. 

Other developers ([Peter Zijlstra](<https://lwn.net/Articles/711244/>) and [Michal Hocko](<https://lwn.net/Articles/711245/>)) have expressed concerns about this behavior as well. Zijlstra asked control-group maintainer Tejun Heo for his thoughts on the matter, but no such thoughts have been forthcoming as of this writing. Starovoitov seems convinced that the current semantics are not problematic, and that they can be changed in some (unspecified) way without breaking compatibility in the future. 

Lutomirski's final worry is a bit more nebulous. Until now, control groups have been concerned with resource control; the addition of BPF filters changes the game. These programs could be another way for an attacker to run hostile code; they could, for example, interfere with the input to a setUID program, leading to potential privilege escalation issues. The programs could also stash useful information where an attacker could find it. 

This sounds a lot like seccomp with a narrower scope but a much stronger ability to exfiltrate private information. 

Unfortunately, while seccomp is very, very careful to prevent injection of a privileged victim into a malicious sandbox, the CGROUP_BPF mechanism appears to have no real security model. There is nothing to prevent a program that's in a malicious cgroup from running a setuid binary. 

For now, attaching a network filter program is a privileged operation, so exploits are not an immediate concern. But as soon as somebody tries to make it work within user namespaces a whole new can of worms would be opened up. Lutomirski put out a "[half-baked proposal](<https://lwn.net/Articles/711246/>)" that would prevent the creation of "dangerous" control groups (those that have filter programs attached) unless various conditions were met to prevent privilege escalation issues in the future. 

That proposal has not met with a lot of approval. Once again, such restrictions would need to be imposed from the outset to limit the risk of breaking systems in the future; that would imply that this feature would need to be disabled for the 4.10 release. But there seems to be little interest in doing that; while Starovoitov [agreed](<https://lwn.net/Articles/711247/>) early on that there was work to be done in the security area, he once again said that it could be done at some future point. 

That is where the discussion stands, as of this writing. If no action is taken, 4.10 will be released with a new feature despite the existence of concerns about its ABI and security. History has some clear lessons about what can happen when new ABIs are shipped with this kind of unanswered question; indeed, one need not look beyond control groups for examples of the kinds of problems that can be created. Given the probable outcome here, one can only hope that the BPF developers are correct that some way can be found to address the semantic and security issues without creating ABI compatibility problems.
