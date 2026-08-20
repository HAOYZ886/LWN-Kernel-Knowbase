---
title: A reworked BPF API
url: https://lwn.net/Articles/606089/
date: "July 23, 2014"
category: BPF
author: "By Jonathan Corbet July 23, 2014"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 23, 2014

Regular LWN readers will be, by now, well aware of the fact that the kernel's Berkeley Packet Filter (BPF) virtual machine is in the middle of a rapid development phase, moving beyond packet filtering into a number of other roles. "[Extending extended BPF](<https://lwn.net/Articles/603983/>)," published at the beginning of July, covered many of the changes that are in the works for an upcoming kernel release. The patch set covered there has evolved considerably since the article was written; the basic functionality is the same, but the API is not. So another look seems warranted. 

The [version 2 patch set](<https://lwn.net/Articles/605907/>) posted by Alexei Starovoitov retains many of the features of the first version. It still adds a single `bpf()` system call providing a number of new functions. Among those are the ability to load BPF programs, of course, though there is still no way to directly run these programs from user space. In the old patch set, though, BPF programs were represented by numeric IDs in a global namespace. That feature is now gone. Instead, the new interface to load a program looks like this: 

```
int bpf(BPF_PROG_LOAD, enum bpf_prog_type prog_type, struct nlattr *attr,
                int attr_len);
```

As before, there is only one "program type" defined: `BPF_PROG_TYPE_UNSPEC`, and the actual program is to be found in the `attr` array. That array, as before, must also contain an attribute describing the license that applies to the loaded program. Unlike the previous version, version 2 does not prohibit the loading of non-GPL-compatible programs. It does, however, allow functions "exported" to BPF programs from the kernel to be marked GPL-only; non-GPL-compatible programs that attempt to call such a function will fail to load. 

The `attr` array can also contain a special "fixup" section; this feature will be discussed momentarily. 

What's missing from the above call, relative to the first version, is the `prog_id` parameter specifying the global ID number to use. There is no longer any need for an application to specify such an ID; instead, the kernel tracks programs internally and, whenever a program is loaded, an associated file descriptor is allocated and returned to user space. The application can then use that descriptor to refer to the loaded BPF program, which will remain in the kernel for as long as the file descriptor is held open. There is, thus, no longer any need for an explicit "unload program" operation; instead, the application need only close the file descriptor. 

The "maps" feature from version 1 has also been retained, but, again, the global ID numbers are gone. When a map is created (using the `BPF_MAP_CREATE` command to the `bpf()` system call), a file descriptor is once again returned to the calling process. That descriptor can be used to store values in the map, query values, iterate through the map, and so on. Once again, the map will continue to exist for as long as the file descriptor remains open. 

The removal of the global program and map ID namespaces eliminates a whole set of potential problems, including excessive resource usage if programs leave loaded BPF resources lying around after they exit and possible ID number conflicts. In the end, global IDs are reminiscent of the System V IPC API, and that is not something that everybody wants to be reminded of. But it does raise an interesting question: how do loaded BPF programs refer to maps? 

In the previous version of the patch set, using a map to communicate between a BPF program and user space was straightforward; the two sides just had to agree on the proper ID number(s). In the absence of a global ID, an application can refer to a BPF map using the file descriptor passed back from the kernel. But file descriptors only have a meaning in the context of a specific process, and BPF programs do not run in any sort of process context. So the file descriptor cannot be used on the BPF side. 

Alexei's solution is to add a "fixup array" to the process of loading a BPF program. This array contains one or more instances of this structure: 

```
struct bpf_map_fixup {
    	int insn_idx;
    	int fd;
        };
```

The array is passed to the `BPF_PROG_LOAD` operation in the `attr` argument. As part of the loading process, the kernel will iterate through the array. For each entry, `insn_idx` is expected to be the offset within the program of a function call instruction that makes use of a BPF map; the actual map to be passed to that function is represented by `fd`. The program loader will convert `fd` into an internal representation that is available to BPF programs, then modify the indicated instruction accordingly. In this way, the process-specific file descriptor numbers are removed from the program, replaced by internal IDs. 

This solution may strike some readers as being a bit inelegant. For the most part, the BPF virtual machine knows nothing about maps; they are implemented using the "external function" mechanism. Indeed, for the map functionality to be available in any specific context (when running socket filter programs, for example), the kernel code setting up that context must include a fair amount of boilerplate code exporting the map functions to BPF programs. This design allows maps to be implemented with no direct support from the virtual machine; there are no map-specific BPF instructions, for example. 

The addition of the fixup array wrecks that separation, pushing an awareness of maps (and how they are represented) into the core of the BPF program loader. This solution works, but one can't help but wonder if it might not be better just to implement map operations directly as BPF instructions. Then the program loader could recognize those instructions and replace the file-descriptor numbers automatically; user-space programs would not have to track the index of every operation that uses maps and set up a proper fixup array operation. 

As of this writing, though, nobody else has raised such an objection; commentary on this version of the patch set has been quiet in general. That silence suggests that, as a whole, reviewers are relatively happy with what is there. Unless something changes in the near future, it seems likely that a version of this patch set will be put forward for the 3.17 merge window.
