---
title: "The BPF system call API,  version 14"
url: https://lwn.net/Articles/612878/
date: "September 24, 2014"
category: BPF
author: "By Jonathan Corbet September 24, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 24, 2014

Things happen quickly in the Berkeley Packet Filter (BPF) world. LWN last [looked at this work](<https://lwn.net/Articles/606089/>) in July, when version 2 of the patch set adding the `bpf()` system call had been posted. Two months later, this work is up to [version 14](<https://lwn.net/Articles/612764/>); quite a bit has been changed and some functionality has been removed in an attempt to make the patches small enough for reviewers to cope with. At this point, though, the core system call may be reaching a point where it is getting close to ready for entry into the mainline. It seems like a good time for another look at this significant addition to the kernel's functionality, with fervent hopes that it doesn't change yet again. 

BPF developer Alexei Starovoitov has certainly been energetic in his efforts to get this work in condition for merging — the posting of twelve versions in two months, many with significant changes, testifies to that. He has been responsive to requests for changes, but, as [this complaint](<https://lwn.net/Articles/612886/>) suggests, some developers have found him to be a little too pushy. That has not stopped some of his work from getting into the mainline, though, and, in the end, should not be a real impediment to the eventual merging of the rest. 

As with previous versions, the BPF functionality is accessed by way of a single multiplexor system call, but that call has changed significantly: 

```
#include <linux/bpf.h>
    
       int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

The key change, made following [a suggestion from Ingo Molnar](<https://lwn.net/Articles/612882/>), is to create a single large union type holding all of the possible parameter types for the various operations supported by `bpf()`. How that union is used depends on the specific command given to the system call. 

Most of the operations in the current patch set are concerned with the management of maps — arrays of data that can be shared between a BPF program and user space. The process starts with the creation of a map, done with the `BPF_MAP_CREATE` command. With this command, the system call expects the relevant information to be in this member of the `bpf_attr` union: 

```
struct { /* anonymous struct used by BPF_MAP_CREATE command */
    	__u32             map_type;
    	__u32             key_size;    /* size of key in bytes */
    	__u32             value_size;  /* size of value in bytes */
    	__u32             max_entries; /* max number of entries in a map */
        };
```

The `map_type` field describes the type of the map. The plan is to have a wide range of types, including hashed arrays, ordinary arrays, bloom filters, and radix trees. The current implementation claims to only support the hash type, but even that implementation is missing from the actual submission. The `key_size` and `value_size` parameters tell the code how large the keys and associated values will be, while `max_entries` puts an upper bound on the number of items that can be stored in a map. 

When a call is made to `bpf()` to create a map, everything in the `bpf_attr` union beyond the above structure must be set to zero, and `size` should be the size of the union as a whole. These rules, which apply to all `bpf()` operations, are enforced in the code; the purpose is to allow the addition of more information to this union to support future enhancements to BPF functionality. If new fields are added, newer applications can provide the needed information. Older applications, instead, will have to pass zeroes in those fields, so the right thing will happen. 

Upon successful creation of a map, the return value from `bpf()` will be an open file descriptor which can be used to refer to that map. 

There is a set of commands to operate on individual entries in a map; they all use this structure within the `bpf_attr` union: 

```
struct { /* anonymous struct used by BPF_MAP_*_ELEM commands */
    	__u32             map_fd;
    	__aligned_u64     key;
    	union {
    	    __aligned_u64 value;
    	    __aligned_u64 next_key;
    	};
        };
```

For all operations, `map_fd` is the file descriptor referring to the map to be used, and `key` is a pointer to the key of interest. To store an item in the map, the `BPF_MAP_UPDATE_ELEM` command should be used; in this case, `value` should be a pointer to the data to be stored. To look up an item, use `BPF_MAP_LOOKUP_ELEM`; if the item is present in the map, its value will be stored in the location pointed to by `value`. Items can be deleted with `BPF_MAP_DELETE_ELEM`. 

Iterating through a map is done with `BPF_MAP_GET_NEXT_KEY`; it will return the next key following the provided `key`. The meaning of "next" is dependent on the type of the map. Should the given `key` not be found in the map, `next_key` will be set to the first key in the map, so a typical iteration is likely to be started by calling `BPF_MAP_GET_NEXT_KEY` with a nonsense key. 

Note that there is no command to delete a map. Instead, the program that created the map need only close the associated file descriptor; when all descriptors are closed and no loaded BPF programs reference the map, it will be deleted. 

Loading a BPF program into the kernel is accomplished with the `BPF_PROG_LOAD` command. The relevant structure in this case is: 

```
struct { /* anonymous struct used by BPF_PROG_LOAD command */
    	__u32         prog_type;
    	__u32         insn_cnt;
    	__aligned_u64 insns;     /* 'const struct bpf_insn *' */
    	__aligned_u64 license;   /* 'const char *' */
    	__u32         log_level; /* verbosity level of eBPF verifier */
    	__u32         log_size;  /* size of user buffer */
    	__aligned_u64 log_buf;   /* user supplied 'char *' buffer */
        };
```

Here, `prog_type` describes the context in which a program is expected to be used; it controls which data and helper functions will be available to the program when it runs. `BPF_PROG_TYPE_SOCKET` is used for programs that will be attached to sockets, while `BPF_PROG_TYPE_TRACING` is for tracing filters. The size of the program (in instructions) is provided in `insn_cnt`, while `insns` points to the program itself. The `license` field points to a description of the license for the program; it may be used in the future to restrict some functionality to GPL-compatible programs. 

All programs must pass the BPF verifier as part of the loading process. This verifier is meant to ensure that the program cannot do harm to the system as a whole. It will prevent accesses to arbitrary data, disallow programs that have loops, and more. Should a developer want to know why the verifier is rejecting a program, they can set up a logging buffer of length `log_size`, pointed to by `log_buf`. Actually turning on logging is done by setting `log_level` to a non-zero value. 

Note that the "fixup" array found in early versions of the patch set is no longer present. That array indicated the instructions referring to BPF map file descriptors; said instructions were fixed to use internal pointers by the verifier. Current versions of the patch set, instead, define new BPF instructions for map access. The verifier can recognize those instructions directly, so user space is no longer required to point them out. 

In the v14 patch set, there is no way to actually attach BPF programs to interesting events once they are loaded. Such features are meant to be added once the basic BPF functionality has gotten through review and found its way into the mainline. That point seems to be getting closer; the developers who have taken an interest in the API seem to be increasingly happy with what they have. A 3.18 merge seems ambitious at this point, but 3.19 might be a real possibility. 

**Update** : this series was accepted into the net-next tree on September 26, so it almost certainly will show up in 3.18.
