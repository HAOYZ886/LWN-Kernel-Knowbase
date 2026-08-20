---
title: "dma-buf and binary-only modules"
url: https://lwn.net/Articles/483101/
date: "February 22, 2012"
category: Copyright issues
author: "By Jonathan Corbet February 22, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 22, 2012

The [DMA buffer sharing mechanism](<https://lwn.net/Articles/474819/>) has been merged for the 3.3 kernel; it is a way for DMA buffers to be shared between otherwise independent device drivers under user-space control. The dma-buf patches, as merged for 3.3, include a number of functions used by drivers to access buffers; those functions are all exported in the GPL-only mode. That drew a [complaint](<https://lwn.net/Articles/476007/>) from Robert Morell of NVIDIA, who, unsurprisingly, didn't like the fact that this interface would be unavailable to his company's proprietary driver. 

It will be unsurprising to most readers that the response to Robert's complaint was not 100% sympathetic. After a while, the discussion died down without any real resolution. Recently, though, Rob Clark has [reported](<https://lwn.net/Articles/483104/>) on a discussion held at the Embedded Linux Conference: 

Following the discussion, I agree that dma-buf infrastructure is intended as an interface between driver subsystems. And because (for now) all the other arm SoC gl(es) stacks unfortunately involve a closed src userspace, and since I consider userspace and kernel as tightly coupled in the realm of graphics stacks, I don't think we can really claim the moral high-ground here. So I can't object to use of EXPORT_SYMBOL() instead of EXPORT_SYMBOL_GPL(). 

Since then, there has been no discussion at all; there has also been no move to change the symbol exports in the mainline kernel. But the shift in tone suggests that positions may be softening, and that the buffer-sharing API may eventually be made available to proprietary modules.
