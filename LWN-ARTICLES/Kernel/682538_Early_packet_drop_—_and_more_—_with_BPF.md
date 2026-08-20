---
title: Early packet drop — and more — with BPF
url: https://lwn.net/Articles/682538/
date: "April 6, 2016"
category: "BPF-Networking; Networking-eXpress Data Path XDP"
author: "By Jonathan Corbet April 6, 2016"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
April 6, 2016

The [Berkeley packet filter (BPF) mechanism](<https://lwn.net/Articles/612878/>) has been working its way into various kernel subsystems since it was rewritten and extended in 2014. There is, it turns out, great value in an in-kernel virtual machine that allows for the implementation of arbitrary policies without writing kernel code. A recent patch set pushing BPF into networking drivers shows some of the potential of this mechanism — and the difficulty of designing its integration in a way that will stand the test of time. If it is successful, it may change the way high-performance networking is done on Linux systems. 

#### Early drop

[This patch set](<https://lwn.net/Articles/682290/>) from Brenden Blanco is, in one sense, a return to the original purpose of BPF: selecting packets for either acceptance or rejection. In this case, though, that selection is done at the earliest possible moment: in the network-adapter device driver, as soon as the packet is received. The intent is to make the handling of packets destined to be dropped as inexpensive as possible, preferably before doing any protocol-processing work, such as setting up a `sk_buff` structure (SKB), for those packets. 

BPF programs, as loaded by the `bpf()` system call, have a type associated with them; that type is checked before a program can be loaded for a specific task. Brenden's patch set starts by defining a new type, `BPF_PROG_TYPE_PHYS_DEV`, for programs that will do early packet processing. Each program type includes a "context" for information that is made available when the program runs; in this case, the context needs to include information about the packet under consideration. Internally, that context is represented by `struct xdp_metadata`; it contains only the length of the packet in this version of the patch set. 

The next step is to add a new `net_device_ops` method that drivers can supply: 

```
int (*ndo_bpf_set)(struct net_device *dev, int fd);
```

A call to `ndo_bpf_set()` tells the driver to install the BPF program indicated by the provided file descriptor `fd`; the program should replace the existing program, if any. A negative `fd` value means that any existing program should be removed. There is a new netlink operation allowing user space to set a program on a given network device. 

The driver can use `bpf_prog_get()` to get a pointer to the actual BPF program from the file descriptor. When a packet comes in, the `BPF_PROG_RUN()` macro can be used to run the program on the packet; a non-zero return code from the program indicates that the packet should be dropped. 

#### Just a starting point

The interface for the running of the BPF program is where the disagreement starts. The driver must clearly give information about the new packet to the program being run; that is done by passing an SKB pointer to `BPF_PROG_RUN()`. The internal machinery hides the creation of the `xdp_metadata` information from the passed-in SKB. The mechanism seems straightforward enough, and it takes advantage of the existing BPF functionality for working with SKBs, but there are a couple of objections. The first of those is that the whole point of the early-drop mechanism is to avoid the overhead of packet processing on packets that will be dropped anyway; the initial, and not insignificant, part of that overhead is the creation of the SKB structure. Creating it anyway would appear to be defeating the purpose. 

In truth, the one driver (`mlx4`) that has been modified to implement this mechanism doesn't create a full SKB; instead, it puts the minimal amount of information into a fake, statically allocated SKB. That avoids the overhead, but at the cost of creating an SKB that isn't really an SKB. The amount of information that needs to go into this fake SKB will surely grow over time — there is surprisingly little call for the ability to drop packets using their length as the sole criterion. Whenever new information is needed, every driver will have to be tweaked to provide it, and, over time, the result will look increasingly like a real SKB with the associated overhead. 

The other potential problem is that there is a fair amount of interest in eventually pushing the BPF programs (possibly after a translation pass) into the network adapter itself. That would allow packets to be dropped before they come to the kernel's attention at all, optimizing the process further. But the hardware is not going to have any knowledge of the kernel's SKB structure; all it can see is what is in the packet itself. If BPF programs are written to expect an SKB to be present, they will not work when pushed into the hardware. 

There is a larger issue, though: quickly dropping packets is a nice capability, but high-performance networking users want to do more than that. They would like to be able to load BPF programs to do fast routing, rewrite packet contents at ingress time, perform decapsulation, coalesce large packets, and so on. Indeed, there is a whole vision for the "express data path" (or "XDP") built around low-level BPF packet processing; see [these slides [PDF]](<https://github.com/iovisor/bpf-docs/raw/master/Express_Data_Path.pdf>) for an overview of what the developers have in mind. In short, they want to provide the sort of optimized processing performance that attracts users to user-space networking stacks while retaining the in-kernel stack and all its functionality. 

If the mechanism is to be extended beyond drop/accept decisions, the information and functionality available to BPF programs will clearly have to increase, preferably without breaking any existing users. As Alexei Starovoitov [put it](<https://lwn.net/Articles/682552/>): ""We have to plan the whole project, so we can incrementally add features without breaking abi"". The current patch set does not reflect much planning of this type; it is, instead, a request-for-comments posting introducing the mechanism that the XDP developers want to build on. 

So, clearly, this code will not be going into the mainline in its current form. But it has had the desired effect of getting the conversation started; there is, it would seem, a lot of interest in adding this feature. If the XDP approach is able to achieve its performance and functionality goals, it should give user-space stacks a run for their money. But there is some significant work to be done to get to that point.
