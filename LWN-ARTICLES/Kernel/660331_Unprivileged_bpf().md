---
title: Unprivileged bpf()
url: https://lwn.net/Articles/660331/
date: "October 12, 2015"
category: "BPF-Unprivileged"
author: "By Jonathan Corbet October 12, 2015"
---

By **Jonathan Corbet**  
October 12, 2015

Over the last couple of years, the [Berkeley packet filter (BPF)](<https://lwn.net/Articles/612878/>) in-kernel virtual machine has gained capabilities and moved beyond its origins in the networking subsystem. Among other things, it has gained its own system call — `bpf()` — to enable the loading of BPF programs into the kernel and various ancillary functions. In current kernels, `bpf()` is a root-only system call, and truly root-only at that: one must be root in the initial system namespace to use it. But the plan always included making `bpf()` available to unprivileged users; now patches are circulating to take that last step. 

Kernel developers are understandably nervous about allowing unprivileged users to load code into the kernel for execution in kernel context. It does not take long to think of a number of ways in which things could go wrong. Getting past the initial reflex that says to simply disallow such access requires a high degree of certainty that there is no way for a rogue BPF program to compromise the kernel in any way. 

The job of providing this assurance belongs to the BPF verifier module. It checks that any program presented for loading does not exceed the maximum number of instructions (4096 by default) and that it does not contain any loops, thus ensuring that it will not take excessive time to run. All jumps are checked to be sure that they land within the program (and don't create loops). Accesses to memory are not allowed to go outside the memory area provided by the kernel. The type of data stored in each accessible memory location is tracked by simulating the program's execution; instructions are not allowed to operate on inappropriate data types, and uninitialized memory cannot be accessed. 

According to BPF developer Alexei Starovoitov, there is just one thing that is missing: ensuring that a BPF program cannot leak information about the kernel — and kernel pointers in particular — to user space. This information can be highly useful to an attacker trying to exploit a vulnerability, so quite a bit of effort has gone into plugging such leaks in recent years. BPF programs do not have access to a great deal of kernel information, but a hostile program still might succeed in exfiltrating something that an attacker can use. That makes it dangerous to allow unprivileged users to load and run BPF programs. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Avoiding this problem requires extending the capabilities of the BPF verifier; that is the intent of [this patch set](<https://lwn.net/Articles/660080/>) from Alexei. Since the verifier already knows the data types of the values stored in each memory location, this is a matter of restricting what can be done with locations containing pointer values. So simply returning a pointer to user space is clearly to be disallowed, as is storing a pointer into a BPF map. Perhaps more subtly, comparison of pointers is also disallowed; otherwise a BPF program could arrive at a pointer's value indirectly. 

Another interesting possibility has to do with pointers stored in the stack. A clever program could try to overwrite parts of a pointer with numeric values in a way that makes it possible to recover the original pointer value, then return the resulting numeric value to user space. The verifier clearly has to catch this case and block it. The list goes on, but the basic idea should be clear by now: there are a lot of ways to sneak pointer values out to user space; the verifier must anticipate and catch them all. Alexei's patch set tries to get the verifier to the point that it can meet that challenge. 

Even with these checks in place, there are still some limits on the use of BPF programs by unprivileged users. In particular, with the current patch set, only socket-filter programs can be loaded by those users. BPF programs used in other settings (tracing, for example) inherently have to deal with kernel data, so they will remain restricted to root. BPF programs also pin down some kernel memory; to keep users from occupying too much memory, the space used is charged against their `RLIMIT_MEMLOCK` resource limit. On many systems, the default value of that limit may prove to be too small to load useful programs, so it may need to be increased by the system administrator. 

Finally, there is a sysctl knob (`kernel.unprivileged_bpf_disabled`) that can be used to disable unprivileged access to `bpf()` entirely. It defaults to "false" (in the patch; distributors may choose differently); if it is set to "true" it cannot be reset without a reboot. There was some talk of adding more fine-grained control over what BPF programs from unprivileged users can do, but Alexei [dismissed](<https://lwn.net/Articles/660338/>) that idea as being driven only by "fear." If the verifier is working as intended, he said, there is no reason for such fear and no use for finer-grained controls. 

That does leave open the question of whether the verifier has indeed earned the trust that is being placed in it, or whether perhaps some fear might be justified. It is 2000 lines or so of moderately complex code that has been reviewed by a relatively small number of (highly capable) people. It is, in a real sense, an implementation of a blacklist of prohibited behaviors; for it to work as advertised, all possible attacks must have been thought of and effectively blocked. That is a relatively high bar. There is a reason why David Miller [described](<https://lwn.net/Articles/660489/>) the patch set as ""scary stuff"". 

But David applied the patch set for the 4.4 merge window despite his misgivings for a reason: as with user namespaces, unprivileged BPF access has the potential to increase system security by reducing the number of places where code must be run with elevated privileges. But first it has to get to a point where all of the exploitable loose ends have been found and tied up — and the introduction of new loose ends must be prevented. That point may well be reached in the relatively near future. But it would not be surprising if this feature were to be disabled by distributors for a while after it hits the mainline.
