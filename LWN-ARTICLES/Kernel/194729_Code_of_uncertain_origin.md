---
title: Code of uncertain origin
url: https://lwn.net/Articles/194729/
date: "August 9, 2006"
category: Copyright issues
author: 
---

Recently, a [set of patches](<http://lwn.net/Articles/194368/>) was posted for inclusion in the mainline kernel. These patches make use of the (undocumented) "SMAPI" BIOS found in Thinkpad laptops to provide support for a number of useful Thinkpad features. It looks like it could be the sort of code that would be welcomed; improving hardware support is generally considered to be a good thing to do. 

There is just one little problem. The code was signed off as: 

```
Signed-off-by: Shem Multinymous <multinymous@gmail.com>
```

Various developers quickly pointed out that there was little useful information here, and that code signed off by an obvious pseudonym would be difficult to trust enough to merge into the kernel. "Mr. Multinymous" argued the case for inclusion with [statements](<https://lwn.net/Articles/194731/>) like: 

I hereby declare that this patch was developed solely based on public specifications, observation of hardware behavior by trial&e[r]ror, and specifications made available to me in clean-room settings and with no attached obligations. So this patch is as pure as the mainline hdaps driver it fixes (and probably purer than many other drivers), and not a single line of it is a derivative work of $OTHER_OS code. 

The author of the code remains unwilling to reveal him or herself, however, with the result that others have refused to consider the code for inclusion. The standoff might have been broken by Pavel Machek, who has offered to sign off the code. Whether that is good enough will be decided by Linus, presumably, sometime after he returns from his travels. 

In the post-SCO world, it does not take a great deal of paranoia or imagination to suppose that somebody could attempt to sabotage the kernel project through the deliberate injection of illicit code. If the true nature of the code were revealed after it had been widely shipped, the result could be a great deal of trouble for kernel developers, Linux distributors, and possibly even users. So it is a good thing for the kernel developers to hold the line and not accept code from anonymous posters. The SCO episode has shown the world just how clean the kernel code base is; we would like to keep it that way. 

That said, it is hard to avoid the disquieting feeling that, had this code been posted under a more normal-sounding name, it would not have been subjected to such scrutiny. Code does show up from unknown names from all parts of the world, and nobody has the resources or the desire to verify that those names belong to real people who have a legitimate right to contribute that code. For this reason, people contributing code which demonstrates deep knowledge of undocumented hardware will often be asked just how they came by that knowledge. Verifying the answer can be difficult, however. Our defenses are thin, but it is hard to see how they could be improved without killing the process entirely.
