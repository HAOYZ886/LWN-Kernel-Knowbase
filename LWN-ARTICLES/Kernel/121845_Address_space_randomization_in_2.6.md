---
title: Address space randomization in 2.6
url: https://lwn.net/Articles/121845/
date: "February 2, 2005"
category: "Memory management-User-space layout; Security-Address-space layout randomization"
author: 
---

Arjan van de Ven has posted [a series of patches](<https://lwn.net/Articles/120966/>) which add some address space randomization to the 2.6 kernel. With these patches applied, each process's stack will begin at a random location, and the beginning of the memory area used for `mmap()` (which is where shared libraries go, among other things) will be randomized as well. These patches represent an improvement in the kernel's security infrastructure, but the reception on the public lists has been surprisingly hostile. 

Many buffer overflow exploits, especially those used in large-scale attacks, contain hardcoded addresses. An exploit which overflows a stack variable will place some executable code on the stack; it then overwrites the return pointer so that the broken function "returns" into the exploit code. If you look at a given distribution's shipped version of a vulnerable program, an exploit will always be able to place its payload at the same address on the stack, so it can contain that address directly. If, instead, the exploit author does not know ahead of time where the payload will end up, actually getting the computer to execute that code will be much harder. 

That is why the stack randomization patch helps. When the stack location is deterministic, a relatively simple exploit can be made to work on all systems running the vulnerable distribution. If the stack moves, instead, hardcoded addresses no longer work. 

Moving the `mmap()` area has similar benefits. One popular type of exploit prepares the stack and then "returns" into a shared library somewhere. That return can, for example, cause the application to behave as if it had intentionally called `system()` or a similar library function. Moving the libraries around makes these attacks harder. 

One of the biggest complaints that has been raised is that the amount of randomization is insufficient. The patches, as posted, vary the stack base within a 64KB area and the `mmap()` base within a 1MB range. Alignment requirements prevent just any address from being used with the result that only a relatively small number of possible base addresses exists. So a determined attacker could repeatedly run a hardcoded exploit with some assurance that, within a reasonable amount of time, the stack would land at the right place and the exploit would work. Placing a long series of no-op instructions at the beginning of the payload can also make an exploit more robust when faced with randomization. 

Arjan [responds](<https://lwn.net/Articles/121846/>) that the amount of randomization is not the issue at the moment. He is trying to get the infrastructure into the kernel and tested in a minimally disruptive way; the degree of randomization can be tweaked upward later on. That amount may never get as high as some people would like, at least on 32-bit systems, because it cuts back on the available virtual address space. But it is likely to go up once the developers are convinced that things are working. 

In any case, a larger randomness makes the problem harder, but does not change its fundamental nature. With the ability to keep trying, an attacker will eventually get around any degree of randomization possible on 32-bit systems (64-bit systems are a different story). Thus, [says Ingo Molnar](<https://lwn.net/Articles/121848/>): 

conclusion: stack randomisation (and other VM randomisations) are not a tool against local attacks (which are much easier and faster to brute-force) or against targeted remote attacks, but mainly a tool to degrade the economy of automated remote attacks. 

Randomization is not a magic bullet which solves a wide range of security problems. It does make an attack harder, however, and that can only be a good thing.
