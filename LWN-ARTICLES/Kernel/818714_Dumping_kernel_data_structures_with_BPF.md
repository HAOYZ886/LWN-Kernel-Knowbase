---
title: Dumping kernel data structures with BPF
url: https://lwn.net/Articles/818714/
date: "April 27, 2020"
category: BPF; printk
author: "By Jonathan Corbet April 27, 2020"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 27, 2020

For as long as operating systems have had kernels, there has been a need to extract information from data structures stored within those kernels. Over the years, a wide range of approaches have been taken to make that information available. In current times, it has become natural to reach for BPF as the tool of choice for a variety of problems, and getting information from kernel data structures is no exception. There are two patches in circulation that take rather different approaches to using BPF to dump information from kernel data structures to user space. 

When your editor first encountered paleolithic Unix systems, tools like `ps` would obtain their information by opening `/dev/kmem` and rooting around directly in the kernel's memory space. This approach had the advantage of requiring no direct kernel support, but there were also some disadvantages, including security issues, lack of atomicity in the collection of complex data, and occasionally returning random garbage. This behavior was perhaps acceptable in the early days, but contemporary users have become strangely less tolerant of it. So digging around in kernel memory has long since fallen out of favor. 

In current Linux systems, this problem is solved with a collection of system calls and virtual files in `/proc`, sysfs, debugfs, and beyond. This approach works, but has some challenges of its own. The kernel must be modified whenever the information to be output changes, "debugging" information in debugfs ends up being needed for normal system operations (where debugfs should not be enabled), and changes can be hard to make without breaking existing applications. So there is a natural desire for something more flexible and adaptable. 

#### Structure dumpers

One approach, [posted](<https://lwn.net/ml/netdev/20200415192740.4082659-1-yhs@fb.com/>) by Yonghong Song, is aimed directly at the virtual-file case. In short, it allows the attachment of BPF programs to implement `/proc`-style files for any supported data structure. 

More specifically, it creates a new virtual filesystem that is expected to be mounted at `/sys/kernel/bpfdump`. It is a singleton filesystem, in that it will provide the same contents regardless of how many times (or in how many different namespaces) it is mounted. Kernel subsystems can then create subdirectories in that filesystem to make specific data structures available. For example, in the patch series, the `task` subdirectory is created to export the active `task_struct` structures from the kernel, `bpf_map` will allow traversal through the list of BPF maps, and `netlink` provides information on active netlink connections. 

Then the patch series adds a new type of BPF program called `BPF_TRACE_DUMP`. A program of this type will be called with a pointer to a structure, and is expected to generate the output for user space, which is written using the [seq_file interface](<https://lwn.net/Articles/22355/>). To that end, two new helper functions — `bpf_seq_printf()` and `bpf_seq_write()` — have been added. These programs are loaded into the kernel with the [`bpf()` system call](<http://www.man7.org/linux/man-pages/man2/bpf.2.html>) in the usual manner. 

Finally, the meaning of the `BPF_OBJ_PIN` command, which was originally added to support programs and maps that persist after the file descriptors referring to them are closed, is extended. With this command, a `BPF_TRACE_DUMP` program can be "pinned" to a file created inside a `/sys/kernel/bpfdump` directory. So, for example, if one wanted to create a new process dumper called "`myps`", one could load a BPF program to generate the desired output from the task structure, then "pin" it to a file named `myps` under `/sys/kernel/bpfdump/task`. 

The patch set includes a few sample programs to demonstrate the mechanism and for self testing; as an example, one can be pinned under `/sys/kernel/bpfdump/netlink` that generates output identical to that from `/proc/net/netlink`. Of course, replicating existing interfaces is not particularly interesting, but it does show how new interfaces can be created. With this capability, users can create interfaces that provide exactly the information they need in a relatively efficient manner. If new information is needed, it can be had without changing the kernel. 

That said, there is some setup required; each structure type that is to be made available in this way requires a certain amount of support code to iterate through active structures and pass them to the relevant BPF program. But that is a one-time effort for each type; after that, in theory, kernel developers need never worry about exporting information from that structure type to user space again. At least, as long as nobody worries that some of the data that is being made available should, instead, be kept secret within the kernel. 

#### `printk()`

The other approach, [posted](<https://lwn.net/ml/linux-kernel/1587120160-3030-1-git-send-email-alan.maguire@oracle.com/>) by Alan Maguire, is oriented more toward debugging needs. When addressing that particular use case, it's only natural to fall back on `printk()` to get information out to user space. 

When debugging a problem, one commonly needs to look at various fields within a kernel data structure. Rebuilding the kernel with a `printk()` call in the right place is usually sufficient to learn something about the issue; often what is learned is that not enough fields were printed and the process needs to start over again. A nice feature to have would be the ability to simply print an arbitrary structure in its entirety; that is often easy to do in interpreted languages like Python, but it is not normally available in C. 

The ability to print specific structure types has existed in the kernel for some time; for example, an [`rtc_time` structure](<https://elixir.bootlin.com/linux/v5.6.7/source/include/uapi/linux/rtc.h#L21>) can be printed directly using the `%ptR` format directive. A relatively small number of structures [is supported](<https://www.kernel.org/doc/html/latest/core-api/printk-formats.html#struct-va-format>), though; each new one requires adding more code to `printk()` and that support must be updated whenever the structure is modified. So this feature is far from a capability to print an arbitrary structure. 

What Maguire realized is that, with the addition of BPF type format (BTF) data to the kernel, it is possible to do something better. BTF was originally added to solve [the problem of BPF program binary portability](<https://lwn.net/Articles/773198/>) between systems. The layout of any given data structure can vary from one kernel configuration to the next, making it hard to create BPF programs that can run universally across all configurations. BTF describes the types used in the kernel as it was actually built; user-space tools can then use that information to "relocate" references within structures to the correct offsets prior to loading a BPF program into the kernel. 

But, once you have a description of a structure's layout available within the kernel, you can use it to print out that structure's data. So Maguire added a new format directive to do so. The format is "`%pT<_type_ >`", where `_type_` is the type of the structure pointer being passed. Making it "`%pT**N** <_type_ >`" adds the field names as well. An example in the patch set prints an `sk_buff` structure (used in the networking layer to hold a packet) with a line like: 

```
pr_info("%pTN<struct sk_buff>", skb);
```

The resulting output looks like this: 

```
{{{.next=00000000c7916e9c,.prev=00000000c7916e9c,
          {.dev=00000000c7916e9c|.dev_scratch=0}}|
          .rbnode={.__rb_parent_color=0,.rb_right=00000000c7916e9c,.rb_left=00000000c7916e9c}|
          .list={.next=00000000c7916e9c,.prev=00000000c7916e9c}},
          {.sk=00000000c7916e9c|.ip_defrag_offset=0},{.tstamp=0|.skb_mstamp_ns=0},
          .cb=['\0'],{{._skb_refdst=0,.destructor=00000000c7916e9c}|
          .tcp_tsorted_anchor={.next=00000000c7916e9c,.prev=00000000c7916e9c}},
          ._nfct=0,.len=0,.data_len=0,.mac_len=0,.hdr_len=0,.queue_mapping=0,
          .__cloned_offset=[],.cloned=0x0,.nohdr=0x0,.fclone=0x0,.peeked=0x0,
          .head_frag=0x0,.pfmemalloc=0x0,.active_extensions=0,.headers_start=[],
          .__pkt_type_offset=[],.pkt_type=0x0,.ignore_df=0x0,.nf_trace=0x0,
          .ip_summed=0x0,.ooo_okay=0x0,.l4_hash=0x0,.sw_hash=0x0,.wifi_acked_valid=0x0,
          .wifi_acked=0x0,.no_fcs=0x0,.encapsulation=0x0,.encap_hdr_csum=0x0,
          .csum_valid=0x0,.__pkt_vlan_present_offset=[],.vlan_present=0x0,
          .csum_complete_sw=0x0,.csum_level=0x0,.csum_not_inet=0x0,.dst_pending_co
```

Here, the original "all on one line" format has been broken up a bit for readability. Output is limited to 1024 characters, which explains the rather abrupt ending seen above. In cases where that limit proves to be a problem, omitting the "`N`" qualifier will allow more fields to be output but without names. Arnaldo Carvalho de Melo [suggested](<https://lwn.net/ml/linux-kernel/20200417164747.GD17973@kernel.org/>) that an additional "`z`" option could suppress the printing of fields whose value is zero, making the output much more compact; that suggestion seems likely to be implemented in the next version of the patch series. 

While `printk()` is the immediate application for this feature, Maguire suggested that it could be used in other settings as well. Ftrace could use it to print out structure contents at tracepoints, for example, or the kernel could use it to enhance the information available in oops listings. 

These patch sets show two different approaches to using the kernel's BPF infrastructure to format information in kernel data structures for use outside of the kernel. They address sufficiently different use cases that it is not a question of which of the two might be accepted; there would appear to be room for both. Each makes it easier to look inside the kernel in its own way.
