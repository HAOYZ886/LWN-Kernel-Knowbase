---
title: "The final wireless extension?"
url: https://lwn.net/Articles/202838/
date: "October 4, 2006"
category: "Development model-User-space ABI; Networking-Wireless; Wireless extensions"
author: 
---

"Wireless extensions" is an `ioctl()`-based API which allows user space to control parameters specific to wireless network interfaces - ESSID, encryption passwords, channels, etc. This API has long been maintained by Jean Tourrilhes; the last few kernel releases have had version 20 of this API. As of this writing, version 21 has been merged into the pre-2.6.19 mainline, but at least some of it may be on its way back out again. 

The problem is that version 21 is a real API change, in that sufficiently old tools will no longer operate properly. In particular, the formatting of the ESSID passed into the kernel has changed, so configurations which associated with a given network under version 20 will not do so under version 21\. There is a workaround (add a space to the ESSID string), but many users will not know that, and, in any case, will only discover the need after upgrading their kernel and finding that the network is no longer there. 

Since this problem came to light, many kernel developers ([including Linus](<https://lwn.net/Articles/202839/>)) have made it clear that they see this sort of API breakage as unacceptable. So they want the ESSID change backed out. There are, of course, real reasons for that change - the way those strings are handled in the protocols has evolved over time. But the right solution is to add a new `ioctl()` which can handle the new string format; the older version would continue to be supported indefinitely. Done in this way, the format change would be acceptable. 

That seems like a good solution, except for one little hitch. It seems that Jean has foreseen this problem for some time. To help minimize the pain, he has been shipping versions of the wireless tools which understand the version 21 API for about six months. A number of distributors have picked up - and shipped - these new tools; affected distributions include Slackware 11 and Mandriva 2007\. If those tools see a wireless extensions version greater than 20, they expect to use the new ESSID string format; if that change is backed out, those tools will break. 

So wireless extensions 21 is now guaranteed to break some systems whether the ESSID change is included or not. At this point, the only way to avoid breaking deployed systems is to keep the wireless extensions version at 20 indefinitely. The wireless extensions, it seems, may be extended no more. 

If that is how things work out, there will be some short-term pain, since needed enhancements will not find their way into the API. The long-term plan, however, is to replace the wireless extensions anyway; to that end, a new, netlink-based API called nl80211 is under development. That API, however, is tightly tied to the Devicescape 802.11 stack, which has been taking rather longer than expected to reach a state where it can be considered for merging. So the Linux wireless API may be stuck for a little while.
