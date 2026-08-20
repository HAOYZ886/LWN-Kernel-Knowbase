---
title: ARM pointer authentication
url: https://lwn.net/Articles/718888/
date: "April 5, 2017"
category: "Security-Security technologies"
author: "By Jonathan Corbet April 5, 2017"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 5, 2017

Many exploits come down to convincing code (kernel or otherwise) to access a pointer that was crafted by the attacker. Buffer-overflow exploits and return-oriented programming, for example, both rely on placing a pointer where a return address is expected; when the processor "returns" to that address, the attacker takes control. Much of the hardening work over the years has focused on making it harder to overwrite addresses in this way. But, as demonstrated by a recent kernel patch set, there may be another way: using a new ARM processor feature to detect and reject crafted addresses. 

In particular, the ARM 8.3 architecture added a feature called "pointer authentication"; its purpose is to detect pointers created by an external entity. In essence, it attaches a cryptographic signature to pointer values; those signatures can be verified before a pointer is used. An attacker, lacking the key used to create the signatures, is unlikely to be able to create valid pointers for use in an exploit. 

Contemporary processors use 64-bit pointer values, but not all of those bits are actually significant. On an ARM64 Linux system using three-level page tables, only the bottom 40 bits are used, while the remaining 24 are equal to the highest significant bit — the 40-bit address is sign-extended to 64 bits, in other words. (For the curious, [Documentation/arm64/memory.txt](<https://lwn.net/Articles/718895/>) describes the virtual address space layout on ARM64 systems). Those uppermost bits (or a subset thereof) could be put to other uses, including holding an authentication code. 

That code is calculated from three values: the pointer itself, a secret key hidden in the process context, and a third value like the current stack pointer. The secret key is intended to make it impossible for an attacker to generate valid codes, while the stack pointer (or some other environmental value) can help prevent the reuse of a valid, signed pointer should one leak to the attacker. The new `PAC` instruction can be used to calculate the authentication code and insert it into a pointer value. 

The value containing the authentication code cannot be dereferenced directly, since, without the sign-extension bits, it is no longer recognized as a valid address. Regaining a usable pointer requires using the `AUT` instruction, which will recalculate the authentication code and compare it to what is found in the authenticated pointer value. If the two match, the authentication code will be removed; otherwise, the pointer will be modified to ensure a fault should it be dereferenced. Thus, any attempt to use a pointer that lacks a proper authentication code will lead to a crash. 

ARM 8.3 provides five separate keys that can be used to authenticate pointers: two for executable (instruction) pointers, two for data, and one "general" key. The [RFC patch set](<https://lwn.net/Articles/718778/>) from Mark Rutland only uses one of the instruction keys, though, reserving the other keys for future use. For the time being, the feature is only provided for user space; it is not yet used within the kernel itself. Whenever a process is created, the kernel will generate a random key and store it in that process's context; the process will then be able to use that key to sign and authenticate pointers, but it cannot read the key itself. 

Actually making use of this feature to, for example, block buffer-overflow exploits is left to user space. The good news here is that the [GCC 7](<https://gcc.gnu.org/gcc-7/changes.html>) compiler will include basic support for pointer authentication in the form of the `-msign-return-address` option. Turning this option on will cause code to be added to function prologues to sign the return address; that address will then be authenticated before returning to it. Options exist to limit authentication to non-leaf functions (those that call other functions), or to all functions in the compilation unit. 

If this feature works as advertised, this return-address authentication should be enough to block basic buffer-overrun attacks. An attacker may be able to overwrite a function's return address, but they cannot generate the proper authentication code, so a jump to that address will never be taken. The code itself is not large, so the potential for brute-force attacks exists, but those attacks cannot be performed without causing the target process to crash multiple times — an outcome that should attract attention in most settings. 

The patch posting is a first-round request for comments, so it is likely to see some changes before being considered for merging. There is some room for future work, including deciding what to do with the other available key values and, perhaps, protecting the kernel as well. There are ways this feature could be used beyond protecting return addresses. Structures containing function pointers are a common target, for example; these, too, could be protected using authentication. Pointer authentication will not solve all of our security problems but it will, with luck, make our systems that much more resistant to attack. 

(More information about this feature can be found in [this Qualcomm white paper [PDF]](<https://www.qualcomm.com/media/documents/files/whitepaper-pointer-authentication-on-armv8-3.pdf>).
