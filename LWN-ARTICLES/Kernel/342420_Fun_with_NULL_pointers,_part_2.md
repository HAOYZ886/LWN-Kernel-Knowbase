---
title: "Fun with NULL pointers, part 2"
url: https://lwn.net/Articles/342420/
date: "July 21, 2009"
category: "Security-Vulnerabilities"
author: "By Jonathan Corbet July 21, 2009"
---

By **Jonathan Corbet**  
July 21, 2009

[Fun with NULL pointers, part 1](<http://lwn.net/Articles/342330/>) took a detailed look at the long chain of failures which allowed the kernel to be compromised by way of a NULL pointer dereference. Eliminating that particular bug was a straightforward fix; it was, in fact, fixed before the nature of the vulnerability was widely understood. The importance of this particular problem is, in one sense, relatively small; there are very few distributions which shipped vulnerable versions of the kernel. But this exploit suggests that there could be a whole class of related problems in the kernel; there is a definite chance that similar vulnerabilities could be discovered - if, indeed, they have not already been found. 

One obvious problem is that ~~when the security module mechanism is configured into the kernel, the administrator-specified limits on the lowest valid user-space virtual address are ignored~~ security modules are allowed to override the administrator-specified limit (`mmap_min_addr`) on the lowest valid user-space address. This behavior is a violation of the understanding by which security modules operate: they are supposed to be able to restrict privileges, but never increase them. In this case, the mere presence of SELinux increased privilege, and the policy enforced by most SELinux deployments failed to close that hole (comments in the exploit code suggest that AppArmor fared no better). 

Additionally, with security modules configured out entirely, `mmap_min_addr` was not enforced at all. The mainline now has a patch which causes the `map_min_addr` sysctl knob to always be in effect; this patch has also been put into the 2.6.27.27 and 2.6.30.2 updates (as have many of the others described here). 

Things are also [being fixed at the SELinux level](<http://blog.namei.org/2009/07/18/a-brief-note-on-the-2630-kernel-null-pointer-vulnerability/>). Future versions of Red Hat's SELinux policy will no longer allow unconfined (but otherwise unprivileged) processes to map pages into the bottom of the address space. There are [still some open problems](<http://lwn.net/Articles/342573/>), though, especially when programs like WINE are thrown into the mix. It's not yet clear how the system can securely support a small number of programs needing the ability to map the zero page. Ideas like running WINE with root privilege - thus, perhaps, carrying Windows-like behavior a little too far \- have garnered little enthusiasm. 

There is another way around `map_min_addr` which also must be addressed: a privileged process which is run under the SVR4 personality will, at `exec()` time, have a read-only page mapped at the zero address. Evidently some old SVR4 programs expect that page to be there, but its presence helps to make null-pointer exploits possible. So another patch merged into mainline and the stable updates resets the SVR4 personality (or, at least, the part that maps the zero page) whenever a setuid program is run. This patch is enough to defeat the pulseaudio-based trick which was used to gain access to a zero-mapped page. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

This change is not enough for some users, who have [requested](<https://lwn.net/Articles/342425/>) the ability to turn off the personality feature altogether. The ability to run binaries from 386-based Unix systems just lacks the importance it had in, say, 1995, so some question whether the personality feature makes any sense given its costs. Linus [answered](<https://lwn.net/Articles/342427/>): 

We could probably get rid of that idiotic feature. It's simply not important enough any more. Does anybody really care? At the same time, over years we've grown _other_ personality flags, and some of them are still relevant. 

In particular, it seems that the ability to disable address-space randomization (which is a personality feature) is useful in a number of situations. So `personality()` is likely to stay, but its zero-page mapping feature might go away. 

Yet another link in the chain of failure is the removal of the null-pointer check by the compiler. This check would have stopped the attack, but GCC optimized it out on the theory that the pointer could not (by virtue of already having been dereferenced) be NULL. GCC (naturally) has a flag which disables that particular optimization; so, from now on, kernels will, by default, be compiled with the `-fno-delete-null-pointer-checks` flag. Given that NULL might truly be a valid pointer value in the kernel, it probably makes sense to disable this particular optimization indefinitely. 

One could well argue, though, that while all of the above changes are good, they also partly miss the point: a quality kernel would not be dereferencing NULL pointers in the first place. It's those dereferences which are the real bug, so they should really be the place where the problem is fixed. There is some interesting history here, though, in that kernel developers have often [been](<http://lkml.org/lkml/2008/2/1/74>) [advised](<http://lkml.org/lkml/2008/2/13/71>) to omit checks for NULL pointers. In particular, code like: 

```
BUG_ON(some_pointer == NULL);
        /* dereference some_pointer */
```

has often seen the `BUG_ON()` line removed with a comment like: 

If we dereference NULL then the kernel will display basically the same information as would a BUG, and it takes the same action. So adding a BUG_ON here really doesn't gain us anything. 

This reasoning is based on the idea that dereferencing a NULL pointer will cause a kernel oops. On its face, it makes sense: if the hardware will detect a NULL-pointer dereference, there is little point in adding the overhead of a software check too. But that reasoning is demonstrably faulty, as shown by this exploit. There are even legitimate reasons for mapping page zero, so it will never be true that a NULL pointer is necessarily invalid. One assumes that the relevant developers understand this now, but there may be a lot of places in the kernel where necessary pointer checks were removed from the code. 

Most of the NULL pointer problems in the kernel are probably just oversights, though. Most of those, in turn, are not exploitable; if there is no way to cause the kernel to actually encounter a NULL pointer in the relevant code, the lack of a check does not change anything. Still, it would be nice to fix all of those up. 

One way of finding these problems may be the [Smatch](<http://smatch.sourceforge.net/>) static analysis tool. Smatch went quiet for some years, but it appears that Dan Carpenter is working on it again; he recently posted [a NULL pointer bug](<https://lwn.net/Articles/342439/>) that Smatch found for him. If Smatch could be turned into a general-purpose tool that could find this sort of problem, the result should be a more secure kernel. It is unfortunate that checkers like this do not seem to attract very many interested developers; free software is very much behind the state of the art in this area and it hurts us. 

Another approach is being taken by Julia Lawall, who has put together a [Coccinelle](<http://lwn.net/Articles/315686/>) "semantic patch" to find and fix check-after-dereference bugs like the one found in the TUN driver. A series of patches ([example](<https://lwn.net/Articles/342440/>)) has been posted to fix a number of these bugs. Cases where a pointer is checked after the first dereference are probably a small subset of all the NULL pointer problems in the kernel, but each one indicates a situation where the programmer thought that a NULL pointer was possible and problematic. So they are all certainly worth fixing. 

All told, the posting of this exploit has served as a sort of wakeup call for the kernel community; it will, with luck, result in the cleaning up of a lot of code and the closing of a number of security problems. Brad Spengler, the author of the exploit, is clearly hoping for a little more, though: he has often expressed concerns that serious kernel security bugs are silently fixed or dismissed as being denial-of-service problems at worst. Whether that will change remains to be seen; in the kernel environment, many bugs can have security implications which are not immediately obvious when the bug is fixed. So we may not see more bugs explicitly advertised as security issues, but, with luck, we will see more bugs fixed.
