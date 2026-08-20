---
title: Regulating wireless devices
url: https://lwn.net/Articles/294675/
date: "August 19, 2008"
category: "Device drivers-Wireless networking; Networking-Wireless"
author: "By Jonathan Corbet August 19, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 19, 2008

Whenever a Linux system communicates with the rest of the world, it must follow a whole set of rules on how that communication is done. Basic TCP/IP networking would work poorly indeed in the lack of an observed agreement on how the networking medium should be used. Wireless networking has all of those constraints, plus a set of its own. Since wireless interfaces are radios, they must conform to rules on the frequencies they can use, how much power they may emit, and so on. If all goes well, Linux will finally have a centralized mechanism for ensuring that wireless devices are operated according to that wider set of rules. 

Regulations on radio transmissions bring some extra challenges. They are legal code, so their violation can bring users, vendors, and distributors into unwanted conversations with representatives of spectrum enforcement agencies. The legal code is inherently local, while wireless devices are inherently mobile, so those devices must be able to modify their behavior to match different sets of rules at different times. And some wireless devices can be programmed in quite flexible ways; they can be operated far outside of their allowed parameters. The possibility that one of these devices could be configured - accidentally or intentionally - in a way which interferes with other uses of the spectrum is very real. 

The potential for legal problems associated with wireless interfaces has cast a shadow over Linux for a while. Some vendors have used it as an excuse for their failure to provide free drivers. Others (Intel, for example), have reworked their hardware to lock up regulatory compliance safely within the firmware. And still, vendors and Linux distributors have worried about what kind of sanctions might come down if Linux systems are seen to be operating in violation of the law somewhere on the planet. Despite all that, the Linux kernel has no central mechanism for ensuring regulatory compliance; it is up to individual drivers to make sure that their hardware does not break the rules. This situation may be about to change, though, as the [Central Regulatory Domain Agent](<http://wireless.kernel.org/en/developers/Regulatory/CRDA>) (CRDA) [patch set](<http://lwn.net/Articles/293820/>), currently being developed by Luis Rodriguez, approaches readiness. 

At the core of CRDA is `struct ieee80211_regdomain`, which describes the rules associated with a given legal regime. It is a somewhat complicated structure, but its contents are relatively simple to understand. They include a set of allowable frequency ranges; for each range, the maximum bandwidth, allowable power, and antenna gain are listed. There's also a set of flags for special rules; some domains, for example, do not allow outdoor operation or certain types of modulation. Each domain is associated with a two-letter identifying code which, normally, is just a country code. 

There is a new mac80211 function which drivers can call to get the current regulatory domain information. But, unless the system has some clue of where on the planet it is currently located, that information will be for the "world domain," which, being designed to avoid offending spectrum authorities worldwide, is quite restrictive. Location information is often available from wireless access points, allowing the system to configure itself without user intervention. Individual drivers can also provide a "location hint" to the regulatory core, perhaps based on regulatory information written to a device's EEPROM by its vendor. If need be, the system administrator can also configure in a location by hand. 

The database of domains and associated rules lives in user space, where it can be easily updated by distributors. When the name of the domain is set within the kernel, an event is generated for udev which, in turn, will be configured to run the `crda` utility. This tool will use the domain name to look up the rules in the database, then use a netlink socket to pass that information back to the kernel. From there, individual drivers are told of the new rules via a notifier function. 

[PULL QUOTE:  No distributors have made any policy plans public, but one assumes that the signing keys for the CRDA database will not be distributed with the system.  END QUOTE] The database is a binary file which is digitally signed; if the signature does not match a set of public keys built into `crda`, then `crda` will refuse to use it. This behavior will protect against a corrupted database, but is also useful for keeping users from modifying it by hand. No distributors have made any policy plans public, but one assumes that the signing keys for the CRDA database will not be distributed with the system. We're dealing with free software, so getting around this kind of restriction will not prove challenging for even moderately determined users, but it should prevent some people from cranking their transmitted power to the maximum just to see what happens. 

The CRDA mechanism, once merged into the kernel and once the wireless drivers actually start using it, should be enough to ensure that Linux systems with well-behaved users will be well-behaved transmitters. Whether that will be enough to satisfy the regulatory agencies (some of which have been quite explicit on their doubts about whether open-source regulatory code can ever be acceptable) remains to be seen. But it is about the best that we can do in a free software environment.
