---
title: Attaching eBPF programs to sockets
url: https://lwn.net/Articles/625224/
date: "December 10, 2014"
category: "BPF-Networking; Packet filtering"
author: "By Jonathan Corbet December 10, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 10, 2014

Recent kernel development cycles have seen the addition of the extended Berkeley Packet Filter (eBPF) subsystem to the kernel. But, as of 3.18, a user-space program can load an eBPF program, but cannot cause it to run in any useful context; programs can be loaded and verified, but then they just sit there. Needless to say, eBPF developer Alexei Starovoitov envisions a more extensive role for this subsystem. The 3.19 kernel should include a new set of patches that will, for the first time, demonstrate the sort of capabilities Alexei has in mind. 

The main feature to be added in 3.19 is the ability to attach eBPF programs to sockets. The sequence of operations is to set up the eBPF program in memory, then use the new (as of 3.18) [`bpf()` system call](<https://lwn.net/Articles/612878/>) to load the program into the kernel and obtain a file descriptor reference to it. Then, the program can be attached to a socket with the new `SO_ATTACH_BPF` option to `setsockopt()`: 

```
setsockopt(socket, SOL_SOCKET, SO_ATTACH_BPF, &fd, sizeof(fd));
```

Where `socket` represents the socket of interest, and `fd` holds the file descriptor for the loaded eBPF program. 

Once the program is loaded, it will be run on every packet that shows up on the given socket. At the moment, the available functionality is still limited in a couple of ways: 

  * eBPF programs have access to the data stored in the packet itself, but not to any other information stored in the kernel's `skb` data structure. Future plans call for making some of that metadata available, but it's not yet clear which data will be reachable or how. 

  * Programs cannot do anything to influence the delivery or contents of the packet. So, while these programs are referred to as "filters," all they can do at the moment is store information in eBPF "maps" for consumption by user space. 

The end result is that eBPF programs will be useful for statistics gathering and such in 3.19, but not a whole lot more. 

Still, that is something to start with. The 3.19 kernel should include a number of examples in the `samples` directory to show how this functionality can be used. Two of them are versions of a simple program that obtains the low-level protocol (UDP, TCP, ICMP, ... ) from each packet and maintains a count for each protocol in an eBPF map. If one wants to write such a program directly in the eBPF virtual machine language, one [ends up](<https://lwn.net/Articles/625230/>) with something like this: 

```
struct bpf_insn prog[] = {
    	BPF_MOV64_REG(BPF_REG_6, BPF_REG_1),
    	BPF_LD_ABS(BPF_B, 14 + 9 /* R0 = ip->proto */),
    	BPF_STX_MEM(BPF_W, BPF_REG_10, BPF_REG_0, -4), /* *(u32 *)(fp - 4) = r0 */
    	BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),
    	BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -4), /* r2 = fp - 4 */
    	BPF_LD_MAP_FD(BPF_REG_1, map_fd),
    	BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),
    	BPF_JMP_IMM(BPF_JEQ, BPF_REG_0, 0, 2),
    	BPF_MOV64_IMM(BPF_REG_1, 1), /* r1 = 1 */
    	BPF_RAW_INSN(BPF_STX | BPF_XADD | BPF_DW, BPF_REG_0, BPF_REG_1, 0, 0),
    	BPF_MOV64_IMM(BPF_REG_0, 0), /* r0 = 0 */
    	BPF_EXIT_INSN(),
        };
```

Needless to say, such programs are, for most of us, not particularly enlightening to to read. But, as is shown in [this example](<https://lwn.net/Articles/625233/>), the program can also be written in a restrictive version of the C language: 

```
int bpf_prog1(struct sk_buff *skb)
        {
    	int index = load_byte(skb, 14 + 9);
    	long *value;
    
    	value = bpf_map_lookup_elem(&my_map, &index);
    	if (value)
    	    __sync_fetch_and_add(value, 1);
    
    	return 0;
        }
```

This program can be fed to a special version of the LLVM compiler, producing an object file for the eBPF virtual machine. For now, one must use Alexei's version of LLVM, but he says that he's working on getting the changes upstreamed into the LLVM mainline. A user-space utility can read the program from the object file and load it into the kernel in the usual way; there is no need to deal directly with the eBPF language. 

The ability to work in a higher-level language makes its value clear when one looks at [the final example](<https://lwn.net/Articles/625238/>), which compiles to a 300-instruction eBPF program. This one does flow tracking, counting the number of packets by IP address. The program itself may be of limited use, but it shows that some fairly complex things can be done with the eBPF virtual machine in the kernel. 

Future plans call for using eBPF in a number of other places, including the secure computing ("seccomp") subsystem and for filtering tracepoint hits. Given that eBPF is becoming a general-purpose facility in the kernel, it seems likely that developers will come up with other places where it can be of use. Expect to see some interesting things happen with eBPF in the coming years.
