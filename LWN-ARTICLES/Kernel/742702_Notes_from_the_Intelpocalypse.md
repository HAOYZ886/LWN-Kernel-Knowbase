---
title: Notes from the Intelpocalypse
url: https://lwn.net/Articles/742702/
date: "January 4, 2018"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet January 4, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 4, 2018

Rumors of an undisclosed CPU security issue have been circulating since before LWN first [covered](<https://lwn.net/Articles/738975/>) the kernel page-table isolation patch set in November 2017. Now, finally, [the information is out](<https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html>) — and the problem is even worse than had been expected. Read on for a summary of these issues and what has to be done to respond to them in the kernel. 

All three disclosed vulnerabilities take advantage of the CPU's speculative execution mechanism. In a simple view, a CPU is a deterministic machine executing a set of instructions in sequence in a predictable manner. Real-world CPUs are more complex, and that complexity has opened the door to some unpleasant attacks. 

A CPU is typically working on the execution of multiple instructions at once, for performance reasons. Executing instructions in parallel allows the processor to keep more of its subunits busy at once, which speeds things up. But parallel execution is also driven by the slowness of access to main memory. A cache miss requiring a fetch from RAM can stall the execution of an instruction for hundreds of processor cycles, with a clear impact on performance. To minimize the amount of time it spends waiting for data, the CPU will, to the extent it can, execute instructions _after_ the stalled one, essentially reordering the code in the program. That reordering is often invisible, but it occasionally leads to the sort of fun that caused [Documentation/memory-barriers.txt](<http://elixir.free-electrons.com/linux/latest/source/Documentation/memory-barriers.txt>) to be written. 

Out-of-order execution runs into a challenge whenever the code branches, though. The processor may not yet be able to tell which branch will be taken, so it doesn't know where to go to execute ahead of the stalled instruction(s). The answer here is "branch prediction". The processor will make a guess based on past experience with the branch in question and, possibly, explicit guidance from the code (the `unlikely()` directive used in kernel code, for example). Once the actual branch condition can be evaluated, the processor will determine whether it guessed right. If not, the "speculatively" executed instructions after the branch will be unwound, and everything will proceed as if they had never been run. 

A branch-prediction failure should really only lead to slower execution, with no visible side effects. That turns out to not be the case, though, leading to a set of severe information-disclosure vulnerabilities. In particular, speculative instruction execution can cause data to be loaded into the CPU memory cache; timing attacks can then be used to learn which instructions were executed. If speculative execution of kernel code can be controlled by an attacker, the contents of the cache can be used as a covert channel to get data out of the kernel. 

#### Getting around boundary checks

Perhaps the nastiest of the vulnerabilities, in terms of the cost of defending against them, allows the circumvention of normal boundary checks in the kernel. Imagine kernel code that looks like this: 

```
if (offset < array1->length) {
            unsigned char value = array1->data[offset];
          	unsigned long index = ((value&1)*0x100)+0x200;
          	if (index < array2->length) // length is < 0x300
                unsigned char value2 = array2->data[index];
         }
```

If `offset` is greater than the length of `array1`, the reference into `array1->data` should never happen. But if `array1->length` is not cached, the processor will stall on the test. It may, while waiting, predict that `offset` is within bounds (since it almost always is) and execute forward far enough to at least begin the fetch of the value from `array2`. Once it's clear that `offset` is too large, all of that speculatively done work will be discarded. 

Except that `array2->data[index]` will be present in the CPU cache. An exploit can fetch the data at both `0x200` and `0x300` and compare the timings. If one is far faster than the other, then the faster one was cached. That means that the inner branch was speculatively executed and that, in particular, the lowest bit of `value` was not set. That leaks one bit of kernel memory under attacker control; a more sophisticated approach could, of course, obtain more than a lowest-order bit. 

If a code pattern like the above exists in the kernel and `offset` is under user-space control, this kind of attack can be used to leak arbitrary data from the kernel to a user-space attacker. It would seem that such patterns exist, and that they can be used to read out kernel data at a relatively high rate. It is also possible to create the needed pattern with a BPF program — some types of which can be loaded and run without privilege. The attack is tricky to carry out, requires careful preparation of the CPU cache, and is processor-dependent, but it can be done. Intel, AMD, and ARM processors are all vulnerable (in varying degrees) to this attack. 

There is no straightforward defense to this attack, and nothing has been merged to date. The only known technique, it would seem, is to prevent speculative execution of code within branches when the branch condition is under an attacker's control. That requires putting in a barrier after every test that is potentially vulnerable. Some [preliminary patches](<https://lwn.net/Articles/742754/>) have been posted to add a new API for sensitive pointer references: 

```
value = nospec_load(pointer, lower, upper);
```

This macro will return the value pointed to by `pointer`, but only if it falls within the given `lower` and `upper` bounds; otherwise zero is returned. There are a number of variants on this macro; see [the documentation](<https://lwn.net/Articles/742757/>) for the full set. This approach is problematic on a couple of counts: it hurts performance, and somebody has to _find_ the vulnerable code patterns in the first place. Current vulnerabilities may be fixed, but there can be no doubt that new vulnerabilities of this type will be introduced on a regular basis. 

#### Messing with indirect jumps

The kernel uses indirect jumps (calling a function through a pointer, for example) frequently. Branch prediction for indirect jumps uses cached results in a separate buffer that only keys on 31 bits of the address of interest. The resulting aliasing can be exploited to poison this cache and cause speculative execution to jump to the wrong location. Once again, the CPU will figure out that it got things wrong and unwind the results of the bad jump, but that speculative execution will leave traces in the memory cache. This issue can be exploited to cause the speculative execution of arbitrary code that will, once again, allow the exfiltration of data from the kernel. 

One rather frightening aspect of this vulnerability is that an attacker running inside a virtualized guest can use it to leak data accessible to the hypervisor — all the data in the host system, in other words. That has all kinds of highly unpleasant implications for cloud providers. One can only hope that those providers have taken advantage of whatever early disclosure they got to update their systems. 

There are two possible defenses in this case. One would be a microcode update from Intel that fixes the issue, for some processors at least. In the absence of this update, indirect calls must be replaced by a [two-stage trampoline](<https://lwn.net/Articles/742756/>) that will block further speculative execution. The performance cost of the trampoline will be notable, which is why Linus Torvalds has [complained](<https://lwn.net/Articles/742764/>) that the current patches seem to assume that the CPUs will never be fixed. There is a set of GCC patches forthcoming to add a flag (`-mindirect-branch=thunk-extern`) to automatically generate the trampolines in cases where that's necessary. As of this writing, no defenses have actually been merged into the mainline kernel. 

#### Forcing direct cache loads

The final vulnerability runs entirely in user space, without the involvement of the kernel at all. Imagine a variant of the above code: 

```
if (slow_condition) {
            unsigned char value = kernel_data[offset];
          	unsigned long index = ((value&1)*0x100)+0x200;
          	if (index < length) 
                unsigned char value2 = array[index];
         }
```

Here, `kernel_data` is a kernel-space pointer that should be entirely inaccessible to a user-space program. The same speculative-execution issues, though, may cause the body of the outer `if` block (and possibly the inner block if the low bit of `value` is clear) to be executed on a speculative basis. By checking access timings, an attacker can determine the value of one bit of `kernel_data[offset]`. Of course, the attacker needs to find a useful kernel pointer in the first place, but a variant of this attack can be used to find the placement of the kernel in virtual memory. 

The answer here is [kernel page-table isolation](<https://lwn.net/Articles/741878/>), making the kernel-space data completely invisible to user space so that it cannot be used in speculative execution. This is the only one of the three issues that is addressed by page-table isolation; it alone imposes a performance cost of 5-30% or so. Intel and ARM processors seem to be vulnerable to this issue; AMD processors evidently are not. 

#### The end result

What emerges is a picture of unintended processor functionality that can be exploited to leak arbitrary information from the kernel, and perhaps from other guests in a virtualized setting. If these vulnerabilities are already known to some attackers, they could have been using them to attack cloud providers for some time now. It seems fair to say that this is one of the most severe vulnerabilities to surface in some time. 

The fact that it is based in hardware makes things significantly worse. We will all be paying the performance penalties associated with working around these problems for the indefinite future. For the owners of vast numbers of systems that cannot be updated, the consequences will be worse: they will remain vulnerable to a set of vulnerabilities with known exploits. This is not a happy time for the computing industry. 

It is, to put it lightly, unlikely that this is the last vulnerability hiding within the processors at the heart of our systems. Like the Linux kernel, these processors are highly complex devices that are subject to constant change. And like the kernel, they probably have a number of unpleasant issues lurking within them. Given that, it's worthwhile to look at how these vulnerabilities were handled; there seems to be some unhappiness on that topic which might affect how future issues are disclosed. It's important to get this right, since we'll almost certainly be doing it again. 

See also: the [Meltdown and Spectre attacks page](<https://spectreattack.com/>), which has a detailed and academic look at these vulnerabilities.
