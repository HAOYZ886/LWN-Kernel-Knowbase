---
title: Reworking the wireless extensions
url: https://lwn.net/Articles/90894/
date: "June 23, 2004"
category: "Networking-Wireless; Wireless extensions"
author: 
---

[Two weeks ago](<https://lwn.net/Articles/87844/>) this page covered the launch of a new wireless networking effort. The scope of this effort now seems to be expanding to a redesign of the "wireless extensions" portion of the network stack. This code handles wireless network interfaces, and, in particular, provides a set of functions to user space for the control of those interfaces. Scott Feldman has posted [an initial set of objectives](<https://lwn.net/Articles/90895/>) for a wireless extensions rework. 

Much of what is being proposed is uncontroversial. There has been some disagreement, however, over proposed changes to the "`iw_handler`" interface. This interface is the mechanism by which wireless adapter drivers respond to `ioctl()` calls from user space. Each driver registers a set of functions, one for each of the command codes supported by the wireless extensions. The mechanism used is different from what is seen in other parts of the kernel, however; a wireless interface driver fills in a simple array of function pointers and passes that to the core. The array is indexed by the `ioctl()` command code, and the proper function is called. 

The problem with this interface is that it defeats the compiler's normal type checking. All wireless extension handler functions must have the same prototype, and there is no real way to tell if the right one is being called. As a way of improving the code base, Jeff Garzik would like to replace the `iw_handler` array with a structure full of specific, named function pointers - the same mechanism which is used in the rest of the kernel. Initially, all of these functions would keep the current `iw_handler` prototype, but, over time, each function would be migrated over to taking exactly the arguments it needs. 

Nobody disputes that the new interface would be cleaner. Jean Tourrilhes, the designer of the wireless extensions, has an objection, however: changing this interface would break backward compatibility. Jean [does not like this idea](<https://lwn.net/Articles/90907/>): 

The wireless extension has remained backward compatible over almost 8 years, while tremendously improving and adding new features. And I believe that moving forward, the price of keeping backward compatibility is small, as you can see from my patch. 

It's possible. It's not difficult. Breaking backward compatibility is not a design goal. 

Jean proposes, instead, to create a wrapper layer around the existing interface, thus avoiding breaking any out-of-tree drivers. Jeff, however, [would rather get rid of the old interface entirely](<https://lwn.net/Articles/90910/>), since he sees it as dangerous. 

We want to design driver interfaces that make it tough for the driver writer to screw up. Excluding yourself, myself, and others on this list, I think we all know that driver writers can't code their way out of a paper bag. A properly designed interface lets the compiler flag incorrect code at the first possible opportunity. 

The other relevant point is that Jeff, like most kernel developers, does not see backward compatibility of internal interfaces as an important goal. Interfaces need to be able to change, and the developers can't be held back by the prospect of breaking out-of-tree drivers. As a result, the wireless extensions changes are quite likely to happen - though, perhaps, not until 2.7.
