---
title: On the alignment of IP packets
url: https://lwn.net/Articles/89597/
date: "June 15, 2004"
category: Internet protocol; Networking; skb reserve
author: 
---

Device drivers for network interfaces must allocate a "socket buffer" ("skb") for each incoming packet. A standard idiom in the skb allocation code is a line like this: 

```
skb_reserve(skb, 2);
```

This call tells the socket buffer code to set aside the first two bytes of the data buffer. The reason why this is done can be seen by looking at the resulting layout of an IP packet in the buffer: 

![\[Packet header layout\]](https://static.lwn.net/images/ns/packet-header.png)

The network stack makes frequent use of the IP addresses stored in the packet. By padding the beginning of an ethernet-style packet by two bytes, a network driver can cause those addresses to be aligned on a four-byte boundary. On some architectures, at least, that alignment will speed access to the addresses and make the networking system faster. 

Or so it might seem. As Anton Blanchard recently [figured out](<https://lwn.net/Articles/89606/>), this padding is not always helpful. A number of modern architectures (Anton works with PPC64, but Intel-style architectures qualify too) have no real problem with unaligned memory accesses, so the two-byte offset on IP packets does not necessarily help things. Unfortunately, the DMA engines in a number of systems _do_ have trouble working with unaligned addresses. A padded packet buffer does not start on an aligned address, with the result that DMA operations to that buffer can be slower than they should be. As network adapters get faster, the DMA performance penalty becomes increasingly significant. 

Anton's proposal was to change the `skb_reserve()` calls into calls to a new `skb_align()` function, which could, depending on the architecture, decide whether to insert the padding or not. David Miller [pointed out](<https://lwn.net/Articles/89610/>), however, that the magic constant "2" appears in quite a few places, and simply removing the padding could create bugs elsewhere in the driver code. 

The real [solution](<https://lwn.net/Articles/89770/>) is likely to be the addition of a defined constant called something like `NET_IP_ALIGN`; this constant would be the amount of padding needed for packet alignment on the current architecture. Yes, things probably should have been done that way from the beginning, but life is like that. In any case, once the constant is in, each individual driver can be looked over and fixed up as need be. And one small obstacle to top performance on high-end network adapters will have been removed.
