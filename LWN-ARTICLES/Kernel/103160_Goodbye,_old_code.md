---
title: "Goodbye, old code"
url: https://lwn.net/Articles/103160/
date: "September 22, 2004"
category: MOD INC USE COUNT; Modules; Netfilter; ipchains
author: 
---

In the Good Old Days, loadable modules had to manage their own reference counts with the `MOD_INC_USE_COUNT` and `MOD_DEC_USE_COUNT` macros. This mechanism was always subject to race conditions; since the count was manipulated inside the module itself, there was no way to avoid situations where the kernel was executing inside the module, but the use count was zero. And that was for correctly written modules; distributing responsibility for the reference count in this way also provided lots of opportunities for module writers to get things wrong. 

So, for 2.6, reference count management was moved up into the code which calls into modules, and the `MOD_*_USE_COUNT` macros were deprecated. In recent times the kernel janitors have been busy, to the effect that, at this point, there are no more users of those macros in the mainline kernel. So Christoph Hellwig has posted [a patch removing them altogether](<https://lwn.net/Articles/102959/>). That patch has not been merged as of this writing, but the writing is clearly on the wall. Any external modules which are still using these macros should probably be fixed up in a hurry. 

Christoph has also [sent out a patch](<https://lwn.net/Articles/102961/>) marking the lightly-used inter_module functions as deprecated. These functions, which perform a sort of run-time linking between modules, have never been seen as elegant or safe to use. 

Rusty Russell, meanwhile, has [added a warning to the kernel](<https://lwn.net/Articles/103161/>) informing users that the ipchains and ipfwadm interfaces to netfilter will be going away soon. They have been obsolete since 2.4, but the kernel developers have kept them around because they are a user-space interface which is still very much in use. Once a site administrator gets a set of firewall rules that works, he or she is rarely amused by the idea of rewriting everything for a new interface. 

Supporting these interfaces requires the maintenance of an intermediate compatibility layer in the netfilter code, however, and that makes maintenance and development of the code hard. In the interests of carrying the code forward, the netfilter developers want to get rid of the older cruft. For now, they are just adding a warning; no time frame has been given for (1) firmer warnings, or (2) actual removal of the code. 

There are a couple of obstacles to actually taking this code out: 

  * The users of the old interfaces. For those trying to convert to iptables, William Stearns has posted [a script which converts ipchains rules to iptables](<https://lwn.net/Articles/103167/>). 

  * 32-bit emulation. The binary interface used by iptables is exceedingly difficult to implement for 32-bit user-space programs in a 64-bit kernel - with the result that it has not been done. For this reason, x86-64 maintainer Andi Kleen has [requested](<https://lwn.net/Articles/103168/>) that ipchains not be removed at this time. Fixing that problem will not be a straightforward task, however. 

In the longer term, it seems clear that the older interfaces have to go. The alternative is a steady accumulation of compatibility cruft which, eventually, causes the kernel to collapse under its own weight.
