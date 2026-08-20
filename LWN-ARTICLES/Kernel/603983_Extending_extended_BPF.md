---
title: Extending extended BPF
url: https://lwn.net/Articles/603983/
date: "July 2, 2014"
category: BPF
author: "By Jonathan Corbet July 2, 2014"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 2, 2014

The Berkeley Packet Filter, or BPF, is a special-purpose virtual machine that was originally developed to support applications that wanted to quickly filter packets out of a stream. Over the years, its use in Linux has grown; back in May, LWN [characterized](<https://lwn.net/Articles/599755/>) BPF as "the universal in-kernel virtual machine." Development on BPF continues; a new patch set adds some interesting capabilities and demonstrates some of what developer Alexei Starovoitov has in mind for this subsystem. 

The first thing [this patch series](<https://lwn.net/Articles/604043/>) does is to move the BPF interpreter out of the networking subsystem. BPF can already be used with non-networking parts of the kernel, and the plans are for such uses to grow over time. So the BPF support code will move into a new subdirectory (`kernel/bpf`) and be maintained independently from the networking code. 

Over the past few development cycles, Alexei has introduced a variant of BPF called "extended BPF" (eBPF) which adds a number of capabilities and performance improvements. Thus far, though, eBPF has only been used within the kernel itself; the existing BPF users load "classic" BPF programs into the kernel which are then translated to eBPF prior to execution. With this patch series, though, eBPF will be made available for direct use from user space. Among other things, that means that the eBPF instruction set will, once users pick it up, become difficult to change. There has been relatively little review of the instruction-set changes so far; anybody who has an interest in how this (significant) addition to the kernel's user-space ABI is defined might want to take a close look in the near future. 

#### Loading programs

The patch series adds a new system call named, simply, `bpf()`; it is a multiplexor for a range of different operations. Alexei also supplies a wrapper library to present those operations as a set of independent functions. Multiplexed system calls have not always been popular with reviewers in the past; if that pattern holds, we may see the multiplexed interface taken out and the various functions implemented directly as separate system calls. 

So, for example, user space can load an eBPF program into the kernel with a call to: 

```
int bpf(BPF_PROG_LOAD, int prog_id, enum bpf_prog_type type, struct nlattr *prog,
                int len);
```

Or, using the wrapper function: 

```
int bpf_prog_load(int prog_id, enum bpf_prog_type prog_type, 
                          struct sock_filter_int *insns, int prog_len, const char *license);
```

In either case, `prog_id` is a number used to identify the program; these numbers exist in a single, global namespace. There is currently only one possible value (`BPF_PROT_TYPE_UNSPEC`) possible for `type`. In the actual system call, the BPF program is found in `prog`; the networking roots of BPF show here, where a netlink attribute is used to hold the code. The length of the attribute array is passed in `len`. The wrapper, instead, hides the `nlattr` structure, but exposes a `struct sock_filter_int` structure (which will likely be renamed in the future) to hold the program. The `license` parameter will be discussed below. 

Naturally, adding the ability to run programs within the kernel brings up a number of interesting security issues. So it not surprising that the biggest part of the patch set is a "verifier" that attempts to ensure that eBPF programs cannot harm the running system. The verifier simulates the execution of the program, looking for problematic behavior. Should something suspect turn up, the program will not be loaded. 

The verifier looks for a number of things. It tracks the state of every eBPF register and will not allow their values to be read if they have not been set. To the extent possible, the type of the value stored in each register is also tracked. Load and store instructions can only operate with registers containing the right type of data (a "context" pointer, for example), and all operations are bounds-checked. The verifier also disallows any program containing loops, thus ensuring that all programs will terminate. 

In this patch set, the `CAP_SYS_ADMIN` capability is required to use any of the `bpf()` system call functions. That restriction may limit interesting future uses of eBPF, but there are a number of potential issues (such as the single global ID namespace and resource use limits) that would have to be dealt with before that restriction could be lifted. 

#### Licensing issues

The `bpf_prog_load()` wrapper also has a `license` parameter; the value passed there is stored in the `nlattr` array prior to the `bpf()` call. It is used to provide a string specifying the license that applies to the eBPF program to be loaded; if that license is not GPL-compatible, the kernel will refuse to load the program. This behavior already appears to be somewhat controversial; reviewers noted that full-blown kernel modules can be loaded (albeit with reduced access) without a GPL-compatible license declaration. It strikes some of them as strange to apply stricter rules to eBPF programs. In response, Alexei has [said](<https://lwn.net/Articles/604047/>) that future revisions might move to a module-like scheme where any program can be loaded but access to some functions might be restricted to GPL-compatible programs. 

There could be some interesting implications from this type of restriction. BPF programs are often generated by other programs; the original BPF, after all, was meant to be emitted by the `tcpdump` tool. One might well wonder what the "source" of such a program actually is. If the Chromium browser generates an eBPF script to define a sandbox for a plugin module, which parts of Chromium, if any, are part of the source for that script? One can imagine that the discussion of this issue could go on for a long time indeed. 

#### Maps

The other significant addition in this patch set is "maps." A map is a simple key/value data store that can be shared between user space and eBPF scripts and is persistent within the kernel. As an example of their use, consider [this simple program](<https://lwn.net/Articles/603984/>) included with the patch set. It creates a map with two entries, indexed by IP protocol type; an eBPF script then inspects passing packets and increments the appropriate entry for each. The program in user space can then query those entries to get a sense for what kind of traffic is passing through the system. 

Maps can only be created or deleted from user space; eBPF programs do not have that capability. Maps are created and deleted with: 

```
int bpf_create_map(int map_id, int key_size, int value_size, int max_entries);
        int bpf_delete_map(int map_id);
```

As with program IDs, the namespace for the `map_id` is shared across the entire system; there is no mechanism to specify which maps a given eBPF (or user-space) program may access. To store values into and retrieve values from maps, user space can call: 

```
int bpf_update_elem(int map_id, void *key, void *value);
        int bpf_lookup_elem(int map_id, void *key, void *value);
        int bpf_delete_elem(int map_id, void *key);
        int bpf_get_next_key(int map_id, void *key, void *next_key);
```

Once again, these are the wrapper functions; the actual operations are done with the `bpf()` system call. On the eBPF side, access to maps is provided with a set of external functions. Interestingly, each place where use of eBPF programs is enabled (see below) must explicitly set up access to the map functions; this access is not provided to eBPF programs by default. Maps, in the end, function both as a persistent data store for eBPF programs and a means for communication with user space. 

#### Running eBPF programs

There is one operation that is conspicuous by its absence in the discussion thus far: the ability to actually _run_ an eBPF program. There is little point in running an eBPF program on demand from user space; there is not much that it could do that couldn't be more easily accomplished directly. Instead, eBPF programs are meant to respond to events within the kernel. 

One common event, of course, is the receipt of a packet from the net. The patch set adds a new form of access to the socket filtering mechanism, allowing a program to directly attach an eBPF program to an open socket: 

```
setsockopt(sock, SOL_SOCKET, SO_ATTACH_FILTER_EBPF, &prog_id, sizeof(prog_id));
```

Here, `prog_id` must be the ID number of a program previously loaded into the kernel with the `bpf()` system call. 

eBPF programs can also be attached to tracepoints; such a program will be run every time that the tracepoint fires. This attachment is done by writing the string "`bpf__ID_` " to the appropriate `filter` file in the tracing debugfs filesystem; once again, `_ID_` is the ID number of a loaded eBPF program. Missing, thus far, is a way to use eBPF directly with the secure computing (seccomp) mechanism; one assumes that will follow at some point. 

All told, this patch set represents a significant addition to the BPF virtual machine. It is also a large addition to the kernel's user-space ABI; that suggests that it needs a rather higher level of review than it has seen so far. Once that review happens, the shape of the patch set may well change from what has been described here. But there seems to be little disagreement that the kernel can benefit from a more capable virtual machine that can be used in a number of contexts. So, sooner or later, some version of these patches will probably go in.
