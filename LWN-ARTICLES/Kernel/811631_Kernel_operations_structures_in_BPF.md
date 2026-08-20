---
title: Kernel operations structures in BPF
url: https://lwn.net/Articles/811631/
date: "February 7, 2020"
category: "BPF-struct ops"
author: "By Jonathan Corbet February 7, 2020"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 7, 2020

One of the more eyebrow-raising features to go into the 5.6 kernel is the ability to load TCP congestion-control algorithms as [BPF](<https://lwn.net/Articles/740157/>) programs; networking developer Toke Høiland-Jørgensen [described it](<https://lwn.net/ml/bufferbloat/87bls8bnsm.fsf@toke.dk/>) as a continuation of the kernel's ""march towards becoming BPF runtime-powered microkernel"". On its face, congestion control is a significant new functionality to hand over to BPF, taking it far beyond its existing capabilities. When one looks closer, though, one's eyebrow altitude may well increase further; the implementation of this feature breaks new ground in a couple of areas. 

The use case for this feature seems clear enough. There are a number of such algorithms in use, each of which is suited for a different networking environment. There may be good reasons to distribute an updated or improved version of an algorithm and for recipients to be able to make use of it without building a new kernel or even rebooting. Networking developers can certainly benefit from being able to play with congestion-control code on the fly. One could argue that congestion control is not conceptually different from other tasks, such as [flow dissection](<https://lwn.net/Articles/764200/>) or [IR protocol decoding](<https://lwn.net/Articles/759188/>), that can be done with BPF now — but congestion control does involve a rather higher level of complexity. 

A look at [the patch set](<https://lwn.net/ml/netdev/20191231062037.280596-1-kafai@fb.com/>) posted by Martin KaFai Lau reveals that what has been merged for 5.6 is not just a mechanism for hooking in TCP congestion-control algorithms; it is far more general than that. To be specific, this new infrastructure can be used to allow a BPF program to replace any "operations structure" — a structure full of function pointers — in the kernel. It is, at this point, only capable of replacing the [`tcp_congestion_ops` structure](<https://elixir.bootlin.com/linux/v5.5/source/include/net/tcp.h#L1043>) used for congestion control; experience suggests, though, that other uses will show up sooner rather than later. 

#### The user-space API

On the user-space side, loading a new operations structure requires a few steps, the first of which is to use the [`bpf()` system call](<http://www.man7.org/linux/man-pages/man2/bpf.2.html>) to load an implementation of each function as a separate BPF program. The new `BPF_PROG_TYPE_STRUCT_OPS` type has been defined for these programs. In the attributes passed with each program, user space must provide the BPF type format (BTF) ID corresponding to the structure being replaced (specifying the actual function being implemented comes later). BTF is a relatively recent addition that describes the functions and data structures in the running kernel; it is currently used for [type-checking of tracing functions](<https://lwn.net/Articles/803258/>) among other purposes. 

User space must also specify an integer offset identifying the function this program will replace. For example, the `ssthresh()` member of `struct tcp_congestion_ops` is the sixth field defined there, so this offset will be passed as five (since offsets start at zero). How this API might interact with [structure layout randomization](<https://lwn.net/Articles/722293/>) is not entirely clear. 

As the programs for each structure member are loaded, the kernel will return a file descriptor corresponding to each. Then, user space must populate a structure that looks like this: 

```
struct bpf_tcp_congestion_ops {
    	refcount_t refcnt;
    	enum bpf_struct_ops_state state;
    	struct tcp_congestion_ops data;
        };
```

The `data` field has the type of the structure to be replaced — `struct tcp_congestion_ops` in this case. Rather than containing function pointers, though, this structure should contain the file descriptors for the programs that have been loaded to implement those functions. The non-function fields of that structure should be set as needed, though the kernel can override things as described below. 

The last step is to load this structure into the kernel. One might imagine a number of ways of doing this; the actual implementation is almost certainly something else. User space must create a special BPF map with the new `BPF_MAP_TYPE_STRUCT_OPS` type. Associated with this map is the BTF type ID of a special structure in the kernel (described below); that is how the map is connected with the structure that is to be replaced. Actually replacing the structure is accomplished by storing the `bpf_tcp_congestion_ops` structure filled in above into element zero of the map. It is also possible to query the map (to see the reference-count and state fields) or to remove the structure by deleting element zero. 

BPF maps have grown in features and capability over the years. Even so, this seems likely to be the first place where map operations have this kind of side effect elsewhere in the kernel. It is arguably not the most elegant of interfaces; most user-space developers will never see most of it, though, since it is, like most of the BPF API, hidden behind a set of macros and magic object-file sections in the `libbpf` library. 

#### The kernel side

Replacing an operations structure requires support in the kernel; there is no ability for user space to replace arbitrary structures at will. To make it possible to replace a specific type of structure, kernel code must create a structure like this: 

```
#define BPF_STRUCT_OPS_MAX_NR_MEMBERS 64
        struct bpf_struct_ops {
    	const struct bpf_verifier_ops *verifier_ops;
    	int (*init)(struct btf *btf);
    	int (*check_member)(const struct btf_type *t,
    			    const struct btf_member *member);
    	int (*init_member)(const struct btf_type *t,
    			   const struct btf_member *member,
    			   void *kdata, const void *udata);
    	int (*reg)(void *kdata);
    	void (*unreg)(void *kdata);
    	const struct btf_type *type;
    	const struct btf_type *value_type;
    	const char *name;
    	struct btf_func_model func_models[BPF_STRUCT_OPS_MAX_NR_MEMBERS];
    	u32 type_id;
    	u32 value_id;
        };
```

There are more details here than can be easily covered in this article, and some of the fields of this structure are automatically filled in by macros. The `verifier_ops` structure has a number of functions used to verify that the individual replacement functions are safe to execute. There is a new field added to that structure in this patch set, `struct_access()`, which regulates which parts, if any, of the operations structure itself can be changed by BPF functions. 

The `init()` function will be called first to do any needed global setup. `check_member()` determines whether a specific member of the target structure is allowed to be implemented in BPF, while `init_member()` verifies the exact value of any fields in that structure. In particular, `init_member()` can validate non-function fields (flags fields, for example). The `reg()` function actually registers the replacement structure after the checks have passed; in the congestion-control case, it will install the `tcp_congestion_ops` structure (with the appropriate BPF trampolines used for the function pointers) where the network stack will use it. `unreg()` undoes that action. 

One structure of this type should be created with a specific name: the type of the structure to be replaced with `bpf_` prepended. So the operations structure for the replacement of a `tcp_congestion_ops` structure is named [`bpf_tcp_congestion_ops`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/bpf_tcp_ca.c#n241>). This is the "special structure" that user space must reference (via its BTF ID) when loading a new operations structure. Finally, a line is added to `kernel/bpf/bpf_struct_ops_types.h`: 

```
BPF_STRUCT_OPS_TYPE(tcp_congestion_ops)
```

The lack of a trailing semicolon is necessary. By virtue of some macro magic and including this file four times into [`bpf_struct_ops.c`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/bpf/bpf_struct_ops.c>), everything is set up without the need of a special function to register this structure type. 

#### In closing

For the curious, the kernel-side implementation of `tcp_congestion_ops` replacement can be found in [`net/ipv4/bpf_tcp_ca.c`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/bpf_tcp_ca.c>). There are two actual algorithm implementations ([DCTCP](<https://git.kernel.org/linus/09903869f69f>) and [CUBIC](<https://git.kernel.org/linus/6de4a9c430b5>)) in the tree as well. 

The ability to replace an arbitrary operations structure in the kernel potentially holds a lot of power; a huge portion of kernel code is invoked through at least one such structure. If one could replace all or part of the [`security_hook_heads` structure](<https://elixir.bootlin.com/linux/v5.5/source/include/linux/lsm_hooks.h#L1831>), one could modify security policies in arbitrary ways, similar to what is proposed with [KRSI](<https://lwn.net/Articles/808048/>), for example. Replacing a [`file_operations` structure](<https://elixir.bootlin.com/linux/v5.5/source/include/linux/fs.h#L1821>) could rewire just about any part of the kernel's I/O subsystem. And so on. 

Nobody is proposing to do any of these things — yet — but this sort of capability is sure to attract interested users. There could come a time when just about any kernel functionality is amenable to being hooked or replaced with BPF code from user space. In such a world, users will have a lot of power to change how their systems operate, but what we think of as a "Linux kernel" will become rather more amorphous, dependent on which code has been loaded from user space. The result is likely to be interesting.
