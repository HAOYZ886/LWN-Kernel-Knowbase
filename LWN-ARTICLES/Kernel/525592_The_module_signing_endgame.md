---
title: The module signing endgame
url: https://lwn.net/Articles/525592/
date: "November 21, 2012"
category: "Modules-Signed; Security-Security technologies"
author: "By Jake Edge November 21, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
November 21, 2012

Inserting a loadable module into the running kernel is a potential security problem, so some administrators want to be able to restrict which modules are allowed. One way to do that is to cryptographically sign modules and have the kernel verify that signature before loading the module. [Module signing](<https://lwn.net/Articles/515007/>) isn't for everyone, and those who aren't interested probably don't want to pay much of a price for that new feature. Even those who are interested will want to minimize that price. While cryptographically signing kernel modules can provide a security benefit, that boon comes with a cost: slower kernel builds. When that cost is multiplied across a vast number of kernel builds, it draws some attention. 

David Miller [complained](<https://plus.google.com/101384639386588513837/posts/aLXZdGRULkP>) on Google+ about the cost of module signing in mid-October. Greg Kroah-Hartman agreed in the comments, noting that an allmodconfig build took more than 10% longer between 3.6 and 3.7-rc1. The problem is the addition of module signing to the build process. Allmodconfig builds the kernel with as many modules as possible, which has the effect of build-testing nearly all of the kernel. Maintainers like Miller and Kroah-Hartman do that kind of build frequently, typically after each patch they apply, in order to ensure that the kernel still builds. Module signing can, of course, be turned off using `CONFIG_MODULE_SIG`, but that adds a manual configuration step to the build process, which is annoying. 

Linus Torvalds noted Miller's complaint and offered up a ""*much* simpler"" solution: [defer module signing](<https://lwn.net/Articles/520259/>) until install time. There is already a mechanism to strip modules during the `make modules_install` step. Torvalds's change adds module signing into that step, which means that you don't pay the signing price until you actually install the modules. There are some use cases that would not be supported by this change, but Torvalds essentially dismissed them: 

Sure, it means that if you want to load modules directly from your kernel build tree (without installing them), you'd better be running a kernel that doesn't need the signing (or you need to sign things explicitly). But seriously, nobody cares. If you are building a module after booting the kernel with the intention of loading that modified module, you aren't going to be doing that whole module signing thing *anyway*. Signed modules make sense when building the kernel and module together, so signing them as we install the kernel and module is just sensible. 

One of the main proponents behind the module signing feature over the years has been David Howells; his code was used as the basis for module maintainer Rusty Russell's [signature infrastructure patch](<https://lwn.net/Articles/515032/>). But, Howells was [not particularly happy](<https://lwn.net/Articles/525884/>) with Torvalds's changes. He would like to be able to handle some of the use cases that Torvalds dismissed, including loading modules from the kernel build tree. He thinks that automatic signing should probably just be removed from the build process; a script could be provided to do signing manually. 

Howells is looking at the signed modules problem from a distribution view. Currently, the keys used to sign modules can be auto-generated at build time, with the public key getting built into the kernel and the private portion being used for signing--and then likely deleted once the build finishes. That isn't how distributions will do things, so auto-generating keys concerns Howells: 

It would also be nice to get rid of the key autogeneration stuff. I'm not keen on the idea of unparameterised key autogeneration - anyone signing their modules should really supply the appropriate address elements. 

That may make sense for distributions or those who will be using long-lived keys, but it makes little sense for a more basic use case. With characteristic bluntness, Torvalds [pointed that out](<https://lwn.net/Articles/525889/>): 

You seem to dismiss the "people want to build their own kernel" people entirely. 

One of the main sane use-cases for module signing is: 

\- CONFIG_CHECK_SIGNATURE=y  
\- randomly generated one-time key  
\- "make modules_install; make install"  
\- "make clean" to get rid of the keys.  
\- reboot.  

and now you have a custom kernel that has the convenience of modules, yet is basically as safe as a non-modular build. The above makes it much harder for any kind of root-kit module to be loaded, and basically entirely avoids one fundamental security scare of modules. 

Kroah-Hartman [agreed](<https://lwn.net/Articles/525908/>) with the need to support the use case Torvalds described, though he noted that keys are not removed by `make clean`, which he considered a bit worrisome. It turns out that `make clean` is documented to leave the files needed to build modules, so `make distclean` should be used to get rid of the key files. 

Russell, who has always been a bit skeptical of module signing, [pointed out](<https://lwn.net/Articles/525912/>) that Torvalds's use case could be handled by just storing the hashes of the modules in the kernel--no cryptography necessary. While that's true, Russell's scheme would disallow some other use cases. Signing provides flexibility, Torvalds [said](<https://lwn.net/Articles/525915/>), and is ""technically the right thing to do"". Russell [countered](<https://lwn.net/Articles/525918/>): 

It's 52k of extra text to get that 'nice flexible'; 1% of my kernel image. That's a lot of bug free code. 

Russell's concerns notwithstanding, it is clear that module signing is here to stay. Torvalds's change was added for 3.7 (with some additions by Russell and Howells). For distributions, Josh Boyer has a [patch](<https://lwn.net/Articles/525943/>) that will add a "`modules_sign`" target. It will operate on the modules in their installed location (i.e. after a `modules_install`), and remove the signature, which will allow the distribution packaging system (e.g. RPM) to generate debuginfo for the modules before re-signing them. In that way, distributions can use Torvalds's solution at the cost of signing modules twice. Since that process should be far less frequent than developers building kernels (or build farms building kernels or ...), that tradeoff is likely to be well worth that small amount of pain.
