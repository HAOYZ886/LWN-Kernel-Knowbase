---
title: Network filtering for control groups
url: https://lwn.net/Articles/698073/
date: "August 24, 2016"
category: "BPF-Networking; Control groups; Networking-Packet filtering"
author: "By Jonathan Corbet August 24, 2016"
---

By **Jonathan Corbet**  
August 24, 2016

Control groups (cgroups) perform two basic functions in the kernel: they allow the hierarchical grouping of processes, and they enable the use of controllers to apply resource limits to the processes in each group. Now there is interest in extending cgroups to allow for the control of network traffic as well, but there is a significant difference of opinion over the best way to implement this control. Naturally, the discussion involves another kernel technology that seems to be spreading out into all areas: the Berkeley packet filter (BPF) virtual machine. 

The objective is to be able to apply a filter to network traffic going to or from any process contained within a given cgroup. The intent may be to improve security, by restricting the traffic that a particular system service or application (contained within its own cgroup) can generate. Or it could be a desire for simple resource control or accounting. Either way, the point is to have this control at the cgroup level, something that the kernel does not support now. 

One possible solution, [posted](<https://lwn.net/Articles/697462/>) by Daniel Mack, is to allow a BPF program to be attached to a cgroup. To that end, the `bpf()` system call is extended with a new `BPF_PROG_ATTACH` operation. Exactly what the program is attached to depends on the type of the program; for now the only type supported is `BPF_PROG_TYPE_CGROUP_SOCKET_FILTER`, but the possibility exists that other types (to make other sorts of policy decisions for cgroups) could be supported in the future. Programs may be attached as either an ingress or an egress filter, controlled by a flag passed to the `bpf()` call. Naturally, there is also a `BPF_PROG_DETACH` operation to remove a BPF program from a cgroup. 

Once the program is attached, it will be run on each packet sent to or from a process in the cgroup, depending on how it was attached — though only the ingress side is implemented in the current patch set. If the program returns one, the packet will be allowed to pass; otherwise it will be dropped. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The idea is thus relatively straightforward; it is similar to the socket filters that an individual process can apply to a socket it owns now. Cgroup maintainer Tejun Heo had some quibbles with the implementation, but had no real objection to the overall design. It seems like something that could be added without a whole lot of trouble — except that one developer has different ideas. 

That developer is Pablo Neira Ayuso, the maintainer of the netfilter subsystem. Perhaps unsurprisingly, he [thinks that the proper solution is based on netfilter](<https://lwn.net/Articles/698080/>) rather than BPF; in particular, he would like to see the establishment of a special table of rules that could be attached to a cgroup. In his opinion, a set of rules that can be queried with existing tools would be easier for administrators to deal with than a relatively opaque BPF program. Multiple sets of netfilter rules can be composed, while the BPF approach only allows for a single program to be attached to a cgroup, limiting flexibility in situations where more than one entity wants to add filtering rules. A netfilter-based approach could also take advantage of the connection tracking that, likely, is already being done, speeding the processing of most packets. Those reasons, he says, make netfilter the better tool for this particular job. 

Daniel [acknowledged](<https://lwn.net/Articles/698085/>) the downsides of the BPF implementation, though he was less convinced about the importance of some of them. It seems that this project was looking at a netfilter-based solution early on, but chose to refocus on BPF. There were concerns that the netfilter developers did not actually want a cgroup-level hook, and that the performance of the netfilter system might not be up to the task. He summarized things this way: 

The whole 'eBPF in cgroups' idea was born because through the discussions over the past months we had on all this, it became clear to me that netfilter is not the right place for filtering on local tasks. I agree the solution I am proposing in my patch set has its downsides, mostly when it comes to transparency to users, but I considered that acceptable. After all, we have eBPF users all over the place in the kernel already, and seccomp, for instance, isn't any better in that regard. 

Even so, he said, he would be willing to look again at a solution based on netfilter, especially if Pablo were willing to help with the implementation — something that Pablo said he could do. BPF developer Alexei Starovoitov was [rather less impressed](<https://lwn.net/Articles/698086/>), suggesting that a netfilter-based solution should be considered as a separate facility in the future, if a way can be found to implement it without slowing things down too much. 

And that is where the discussion stands as of this writing. In a sense, netfilter and BPF were always destined to come into conflict at some point; both are, in essence, mechanisms for loading packet-filtering policy into the kernel. Even if this particular disagreement is solved without undue drama, this question is likely to come up again in other contexts. Thus far, there seem to be few bounds on places where BPF may be applicable but, perhaps, it still isn't the solution to every policy problem that comes along.
