---
title: The return of authoritative hooks
url: https://lwn.net/Articles/273822/
date: "March 18, 2008"
category: "Modules-Security modules; Security-Security modules"
author: "By Jonathan Corbet March 18, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 18, 2008

The containers developers have what would seem to be a relatively straightforward problem: they would like to control access to devices on a per-container basis. Then containers could safely be granted access to specific devices without compromising the overall security of the system - even if a container has a root-capable process which can create new device files. Implementing this feature has been a longer journey than these developers had imagined, though, with the "device whitelist" feature being sent around to different kernel subsystems almost like one of those famous garbage barges from years past. A final resting place may have been found, though, and it may signal a change in how some security decisions are made in the kernel in the future. 

The [original version](<http://lwn.net/Articles/271823/>) of the patch, posted by Pavel Emelyanov, set up a control group for the management of device accessibility within containers. The actual rules - and their enforcement - were stored deep within the device model subsystem. This drew an objection from Greg Kroah-Hartman, who suggested that, instead, this kind of access control should done either with udev or with the Linux security module (LSM) subsystem. Udev does not give the desired degree of control and, apparently, can be problematic for those wanting to run older distributions within containers, so it was not seriously considered. The LSM suggestion was, after some resistance, taken to heart, though. 

The result was the [device whitelist LSM patch](<http://lwn.net/Articles/273208/>), posted by Serge Hallyn. It was a stacking security module which made changes to a number of hooks. This is where James Morris [came in](<https://lwn.net/Articles/273825/>) and suggested that, instead, the whitelist should just be added to the existing capabilities security module. Then there would be no need for a separate module and things could be generally simplified. 

So Serge duly rolled out [version 3 of the patch](<https://lwn.net/Articles/273826/>) which moved the whitelist into the capabilities module. But this one ran into resistance as well. [Quoting James Morris](<https://lwn.net/Articles/273827/>) again: 

Moving this logic into LSM means that instead of the cgroups security logic being called from one place in the main kernel (where cgroups lives), it must be called identically from each LSM (none of which are even aware of cgroups), which I think is pretty obviously the wrong solution. 

Casey Schaufler also [didn't like this idea](<https://lwn.net/Articles/273828/>): 

When the next feature comes along are we going to stuff it into capabilities, too? Maybe we'll cram it into audit or CIPSO instead, but how long can this go on? Eventually we need a mechanism that allows more or less general mix-and-match, maybe with a few rules like "don't mix plaids and stripes" to keep things sane or these lesser facilities have no chance. Seems like we're still making LSM too hard to use 

At this point, the complaint was clearly not with just the device whitelist, but with the capabilities module as well. It seems that capabilities are a bit of a poor fit with the LSM idea as a whole. The fact that they exist at all is a bit of a historical artifact; some developers wanted to see them implemented that way to show the flexibility of the LSM interface and to let capabilities be omitted from embedded setups. As it happens, it's still not possible to remove capabilities, and they impose a bit of a cost on all other security modules. 

The core problem is this: LSM, fundamentally, is a restrictive mechanism. An LSM hook can deny an action, but it can never empower a process to do something it would not have been allowed to do in the absence of the security module. The decision to disallow "authoritative hooks" was [made explicitly back in 2001](<http://lwn.net/2001/1108/kernel.php3>) as a way of restricting the scope of LSM modules and, hopefully, ensuring that those modules would not themselves become security problems. 

But capabilities are an inherently authoritative mechanism - a capability check verifies the existence of a special permission which would otherwise not be there. The device whitelist is the same sort of thing: it grants access which would otherwise be denied. So it fits poorly with the LSM model. 

Serge came back with [yet another patch](<http://lwn.net/Articles/273702/>) which takes the whitelist code out of the LSM framework and, instead, inserts a separate set of hooks into the relevant places in the code. Those hooks sit right next to the LSM hooks, but operate in a permissive manner. So far, this approach seems to be passing muster, with no developers (yet) talking about booting it out into yet another subsystem. 

Things may yet change, though. Casey Schaufler is now [talking about](<https://lwn.net/Articles/273830/>) the creation of a "Linux privilege module" framework for the management of all permissions checks. The normal discretionary access control checks could be moved there, as could all capability and "are they root?" logic. And, of course, the device whitelist code. Nobody has really spoken out against this idea - but, then, nobody has seen any code yet either. But, if things continue in this direction, authoritative hooks may have finally found a home, many years after having been rejected from the LSM mechanism.
