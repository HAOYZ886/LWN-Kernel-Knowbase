---
title: "Lua in the kernel?"
url: https://lwn.net/Articles/830154/
date: "September 9, 2020"
category: "Networking-eXpress Data Path XDP; Packet filtering"
author: "By Jake Edge September 9, 2020 Netdev"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
September 9, 2020

* * *

[Netdev](<https://lwn.net/Archives/ConferenceByYear/#2020-Netdev>)

[BPF](<https://lwn.net/Articles/740157/>) is, of course, the language used for network (and other) customization in the Linux kernel, but some people have been using the [Lua language](<https://www.lua.org/>) for the networking side of that equation. Two developers from [Ring-0 Networks](<https://ring-0.io/en>), Lourival Vieira Neto and Victor Nogueira, came to the virtual [Netdev 0x14](<https://netdevconf.info/0x14/index.html>) to [present that work](<https://netdevconf.info/0x14/session.html?talk-linux-network-scripting-with-lua>). It consists of a framework to allow the injection of Lua scripts into the running kernel as well as two projects aimed at routers, one of which is deployed on 20 million devices. 

Neto introduced the talk by saying that it was also based on work from Ana Lúcia de Moura and Roberto Ierusalimschy of the [Pontifical Catholic University of Rio de Janeiro](<https://en.wikipedia.org/wiki/Pontifical_Catholic_University_of_Rio_de_Janeiro>) (PUC-Rio), which is the home organization of the Lua language. They have been working on kernel scripting since 2008, Neto said, developing the [Lunatik](<https://github.com/luainkernel>) framework for Linux. It allows kernel developers to make their subsystems scriptable with Lua and also allows users to load and run their Lua scripts in the kernel. 

#### Lua and sandboxes

Lua was chosen because it is a small, fast language, he said. It is also widely used as the scripting language in networking tools such as [Wireshark](<https://www.wireshark.org/>), [Nmap](<https://nmap.org/>), and [Snort](<https://www.snort.org/>). The talk focused on scripting two networking subsystems in Linux, netfilter using [NFLua](<https://github.com/luainkernel/nflua>) and the [express data path](<https://lwn.net/Articles/750845/>) (XDP) subsystem with [XDPLua](<https://victornogueirario.github.io/xdplua/>). 

[ ![\[Lourival Vieira Neto\]](https://static.lwn.net/images/2020/netdev-neto-sm.png) ](<https://lwn.net/Articles/830953/>)

It is important that any scripting in the kernel not cause it to malfunction. Scripts should not be able to crash the system, run indefinitely, or corrupt other parts of the system. To ensure that, Lunatik uses the Lua virtual machine (VM) facilities for sandboxing the scripts so that they run in a safe execution environment, he said. 

Lua scripts cannot address memory directly; they can only access it through Lua data types, such as strings and tables. All of the Lua types are allocated by the VM and garbage collected when they are no longer being used. But that is not enough to restrict scripts from causing harm since they could allocate too many objects and use enough memory to harm the rest of the system. A custom memory allocator is used that will cap the amount of memory available to Lua scripts in order to avoid this problem. 

Lua provides "fully isolated execution states", Neto said. Those states are initially created with only the language operators available in them; the developer of the subsystem can then determine which libraries get loaded for additional capabilities given to scripts. Those might be Lua standard libraries or specialized libraries, such as Luadata and LuaRCU; the former provides safe access to data external to the Lua VM, while the latter is a mechanism for sharing data between execution states. Both NFLua and XDPLua use Luadata to access packet data, for example. 

Lua provides a single-threaded execution environment without any primitives, such as mutexes, for synchronization. That means the scripts cannot explicitly block the kernel, but they could still run indefinitely. Lua has a facility to interrupt a script after it has run a certain number of instructions, which is used by both NFLua and XDPLua. Multitasking is allowed by Lunatik via multiple execution states in the kernel. 

Only network administrators with the `CAP_NET_ADMIN` capability can load scripts and access the execution states. [Netlink](<https://man7.org/linux/man-pages/man7/netlink.7.html>) sockets are used to transfer data between the kernel and user space; the capability is checked on each access, he said. 

#### NFLua

NFLua is a [netfilter](<https://www.netfilter.org/>) extension that targets advanced [layer 7](<https://en.wikipedia.org/wiki/OSI_model#Layer_7:_Application_Layer>) (application layer) filtering using Lua. Iptables rules can be applied at [layer 3](<https://en.wikipedia.org/wiki/OSI_model#Layer_3:_Network_Layer>) (network) and [layer 4](<https://en.wikipedia.org/wiki/OSI_model#Layer_4:_Transport_Layer>) (transport) to send packets to NFLua; scripts can then be called to inspect the upper layer. Lua is already widely used by network operators for various tasks, including for security and network monitoring, so Lua is a good fit for this kind of filtering. 

NFLua is implemented as a loadable kernel module that contains the Lunatik framework, the Lua interpreter, and whatever libraries are being made available to execution states. Once it is loaded, the `nfluactl` command can be used to create a Lua state and to load Lua code into it. 

He gave an example of a simple filter based on the User-Agent sent with an HTTP request. An iptables rule is used to direct packets to an execution state and a function in that state by name. Packets matching the rule (being sent to port 80) get passed to NFLua, which calls the named function with the packet data. The function looks up the `User-Agent` from the HTTP request in a table to determine whether to block it or not. The Lua function return value indicates whether netfilter should terminate the connection or allow it to proceed. 

#### XDPLua

At that point, Nogueira took over; he described XDPLua as an extension for XDP that allows using Lua in the data path. It represents the natural evolution of NFLua to process packets before they are handled by the network stack. It creates one Lua execution state per CPU, so it can take advantage of parallelism on modern systems. One of the goals of the project was to add "expressiveness and dynamism" on the data path, so that programmers could create more complex applications to be loaded into the kernel. 

XDP uses BPF, so XDPLua has added wrappers for the Lua C API as BPF helpers, allowing BPF programs to call Lua scripts. The XDPLua developers want BPF and Lua to cooperate, "so we can have the best of both of them", Nogueira said. They wanted the performance of BPF while maintaining the expressiveness of Lua. 

[ ![\[Victor Nogueira\]](https://static.lwn.net/images/2020/netdev-nogueira-sm.png) ](<https://lwn.net/Articles/830952/>)

He quickly went through the same example as Neto. The Lua program is loaded into XDPLua, while a BPF program gets loaded into XDP. When a packet arrives at XDP, the BPF program can call the Lua function to determine whether to block or allow the processing of the request; if it is allowed, then the packet will be passed on to the networking stack. 

Another example that he showed was processing cookie values, which are being used to distinguish bots from legitimate traffic, before the packet ever reaches the web server. On the first request from a particular client, the web server replies with a cookie value and some JavaScript to attach the cookie value to further requests. Since bots typically won't run the JavaScript, they will not have the proper cookie value. 

When a new cookie is generated, the web server will pass its value and the source address to the Lua code, which stores it in a table. The code to actually handle the value is straightforward, simply extracting the cookie value and checking to see that it matches what is in the table. If it is not, the request is dropped before it ever reaches the web server. In addition, the XDP program will add the IP address to its block list so that no further requests will even need to consult the Lua program. 

He also outlined an access-control example using [server name indication](<https://en.wikipedia.org/wiki/Server_Name_Indication>) (SNI) in TLS connection requests to restrict which domains can be connected to. This could be used to disallow local users of a network from accessing a forbidden site. Using a simple block list and function in Lua, along with a BPF program to recognize the TLS client hello message and call the Lua function, the SNI data can be checked from XDP. 

#### Benchmarks

In order to gather some numbers, the access-control example was implemented for NFLua, XDPLua, and XDP (in BPF). The BPF version was difficult to write and turned out to be cumbersome to work with, he said, while the same Lua script is shared between NFLua and XDPLua. [`trafgen`](<https://man7.org/linux/man-pages/man8/trafgen.8.html>) was then used to send TLS client hello packets, with SNI values that were in the block list, as quickly as possible. Two things were measured: how many connections per second are dropped on the server (the drop rate) and the CPU usage. It was a fully virtualized environment, both client and server ran on 8-core 3GHz CPUs with 32GB of RAM each; they were connected by a 10Gbps virtio network interface. 

NFLua could drop roughly 0.5-million packets per second, while both XDPLua and XDP/BPF could handle around three times that rate (1.5Mpps). In addition, XDPLua and XDP/BPF both used roughly 0.1% of the available CPU, while NFLua used 50%. Nogueira said that NFLua only gets the packets once they have gone through the network stack and it does not take advantage of the multiple cores, which may help explain the 500x difference. It is important to note that having XDP call out to Lua did not have a significant impact in terms of CPU usage, he said. 

Neto returned to the video stream to wrap things up before the speakers took questions from attendees; roughly half of the 45-minute slot was devoted to Q&A. He noted that NFLua is used in 20 million home routers and that it is being used by network operators for security and monitoring tasks. The lessons learned from NFLua were incorporated into XDPLua, which is designed from the outset to work cooperatively with BPF, so that developers get the ease of use of Lua combined with the performance of BPF. XDPLua is currently used in Ring-0 Networks firewall products that are deployed as part of the infrastructure at internet point of presence (POP) companies on 10Gbps networks. 

One problem area that they have faced is that XDP does not support extensions as loadable kernel modules. Netfilter supports that functionality, which has been beneficial for developing the Lua-based filtering mechanisms. Maintaining out-of-tree bindings in order to support XDPLua has been somewhat difficult. 

Instead of using an in-kernel verifier, as BPF does, the Lua environments take a sandboxing approach to protect the kernel. The BPF verifier can be hard to work with, as they found when developing the XDP/BPF version of the access-control benchmark, Neto said. With that, they turned it over to questions. 

#### Questions

Tom Herbert, who was shepherding the Netdev track, noted from the outset that it would be an uphill struggle to try to get this work merged into the mainline. The BPF verifier is part of what allows the kernel developers to be comfortable with XDP, so adding Lua to the kernel will require a similar effort to convince them that Lua is also safe. For example, the kernel cannot crash because Lua has accessed memory inappropriately; what is being done to prevent that? Neto reiterated that Lua does not access kernel memory directly—it has no pointer type. It can allocate memory, but that can be (and is) limited. Furthermore, the number of instructions can be limited, so that infinite loops are not possible. 

Neto said that there are various places you can enforce the safety assurances: at compile time, load time, or run time. BPF does load time checks with the verifier, while Lua sandboxes its programs with its VM at run time. There could, of course, be a bug in the VM implementation, but that is also true with the BPF verifier. 

Another question that will likely be asked, Herbert said, is why a Lua-to-BPF compiler could not be created; there are already compilers for C and P4, why not do that for Lua? You could perhaps turn Lua syntax into BPF, Neto said, but you cannot write a Lua VM that runs on BPF, so you wouldn't get all of the features that Lua can provide. The verifier purposely limits the BPF that can be run, so you can't write general-purpose code. You might be able to have some elements of Lua, but not the "full package" if you are targeting BPF. 

Shrijeet Mukherjee said that having two VMs in the kernel was likely to be problematic; he suggested minimizing the Lua VM component in the kernel and to move as much as possible into user space. The BPF VM has momentum and acceptance; from a community perspective, adding another will be difficult. Neto said that getting the Lua work upstream is not necessarily the path being pursued; if XDP could provide a mechanism to allow dynamic extensions, like netfilter has, that could work as well. Herbert said that will be a hard sell; XDP started with the idea that it would be pluggable, but it is now simply a BPF hook. 

Netdev organizer Jamal Hadi Salim said that there is a need for both scripting and compiled code for networking tasks. But there are political and technical hurdles to getting another programming environment added to the kernel. The security concerns are important, but he believes that Lua could meet the requirements, just differently than is done with BPF. 

Mukherjee suggested that there might be a way to split things up, such that the in-kernel packet-handling was done in XDP, while the policy handling could be done with Lua in user space; communication could be done through a shared BPF map. Packet handling is really in the kernel's domain, he said, but the policy aspects may not be. But, as Neto pointed out, that will add latency. They have tried that approach in the past, but the performance was such that they moved on to NFLua and then to XDPLua. 

But Mukherjee wondered if caching the policy decisions in the kernel could avoid much of the added latency for consulting user space. The "basic stuff" could be handled in the kernel, while the "really complicated" pieces are handled in user space—with the results of the decision somehow cached in the kernel. He was not sure that was a reasonable approach, but there may be a middle ground to be found that would still allow much of what Lua is providing without putting it into the kernel. 

An attendee asked about the maturity of XDPLua. Neto said that it is running in production, but it is also still under development. There is no patch ready for upstream submission at this point. There is cleanup work that needs to be done before that can happen. The system used for the benchmarks was overpowered, from a CPU standpoint, for the 10Gbps link speed, so the CPU usage difference between XDP/BPF and XDPLua was not truly shown, an attendee said. Neto agreed that more testing, including using slower virtual CPUs, needs to be done. 

They are using the standard Lua, rather than the [LuaJIT](<https://luajit.org/>) fork, Neto said, in answer to another question. Investigation of a "typed Lua" for compilation is something on the roadmap. That is the approach that the main Lua project is taking to compete with LuaJIT on performance. The Lunatik developers have avoided using LuaJIT directly because it is based on an older version of the language, but they are interested in pursuing the performance gains that could come with compiled and optimized Lua. 

The entrenchment of BPF and its VM make it rather hard to see how Lua could actually be added into the kernel itself. Getting hooks for other pluggable programming environments added to XDP might be a more plausible approach, though Herbert did not seem too optimistic even though he (and others) thought the Lua approach was interesting and potentially useful. But, "it is a moonshot", Herbert said. Whether the XDPLua developers can overcome whatever resistance there will be remains to be seen, but it seems clear that there are at least some who are chafing at the restrictions of the BPF programming environment.
