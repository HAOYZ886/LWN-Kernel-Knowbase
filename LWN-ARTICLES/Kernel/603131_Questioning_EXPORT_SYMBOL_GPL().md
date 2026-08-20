---
title: Questioning EXPORT_SYMBOL_GPL()
url: https://lwn.net/Articles/603131/
date: "June 23, 2014"
category: "Copyright issues; EXPORT SYMBOL GPL; Modules-Exported symbols"
author: "By Jonathan Corbet June 23, 2014"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
June 23, 2014

There have been arguments about the legality of binary-only kernel modules for almost as long as the kernel has had loadable module support. One of the key factors in this disagreement is the `EXPORT_SYMBOL_GPL()` directive, which is intended to keep certain kernel functions out of the reach of proprietary modules. A recent discussion about the merging of a proposed new kernel subsystem has revived some questions about the meaning and value of `EXPORT_SYMBOL_GPL()` — and whether it is worth bothering with at all. 

Loadable modules do not have access to every function or variable in the kernel; instead, they can only make use of symbols that have been explicitly "exported" to them by way of the `EXPORT_SYMBOL()` macro or one of its variants. When plain `EXPORT_SYMBOL()` is used, any kernel module is able to gain access to the named symbol. If the developer uses `EXPORT_SYMBOL_GPL()` instead, the symbol will only be made available to modules that have declared that they are distributable under a GPL-compatible license. `EXPORT_SYMBOL_GPL()` is meant to mark kernel interfaces that are deemed to be so low-level and specific to the kernel that any software that uses them must perforce be a derived product of the kernel. The GPL requires that derived products, if distributed, be made available under the same license; `EXPORT_SYMBOL_GPL()` is thus a statement that the named symbol should only be used by GPL-compatible code. 

It is worth noting that nobody has said that symbols exported with plain `EXPORT_SYMBOL()` can be freely used by proprietary code; indeed, a number of developers claim that all (or nearly all) loadable modules are derived products of the kernel regardless of whether they use GPL-only symbols or not. In general, the kernel community has long worked to maintain a vague and scary ambiguity around the legal status of proprietary modules while being unwilling to attempt to ban such modules outright. 

#### Shared DMA buffers

Recent years have seen a fair amount of development intended to allow device drivers to share DMA buffers with each other and with user space. A common use case for this capability is transferring video data directly from a camera to a graphics controller, allowing that data to be displayed with no need for user-space involvement. The [DMA buffer-sharing subsystem](<https://lwn.net/Articles/474819/>), often just called "dma-buf," is a key part of this functionality. When the dma-buf code was merged in 2012, there was [a lengthy discussion](<https://lwn.net/Articles/483101/>) on whether that subsystem should be exported to modules in the GPL-only mode or not. 

The code as originally written used `EXPORT_SYMBOL_GPL()`. A representative from NVIDIA [requested](<https://lwn.net/Articles/476007/>) that those exports be changed to `EXPORT_SYMBOL()` instead. If dma-buf were to be GPL-only, he [said](<https://lwn.net/Articles/603135/>), the result would not be to get NVIDIA to open-source its driver. Instead: 

For example, in the future, if we can't agree on using EXPORT_SYMBOL, then if somebody were to introduce a laptop that had a Tegra GPU (which uses GPL-compatible open-source Linux drivers) and a GeForce GPU (which is, as described above, supported by our existing binary driver) then I imagine we'd have no choice but to re-implement a different open-source buffer allocation mechanism for Tegra that could be used between the two, or just continue using our existing nvhost code. This, along with every other SoC's version, is exactly what the dma-buf project was intended to replace. 

At the time, a number of the developers involved evidently [discussed the question](<https://lwn.net/Articles/603136/>) at the Embedded Linux Conference and concluded that `EXPORT_SYMBOL()` was appropriate in this case. Other developers, however, [made it clear](<https://lwn.net/Articles/603229/>) that they objected to the change. No resolution was ever posted publicly, but the end result is clear: the dma-buf symbols are still exported GPL-only in current kernels. 

#### On the fence

More recently, a major enhancement to dma-buf functionality has come along in the form of the [fence synchronization](<https://lwn.net/Articles/602637/>) subsystem. A "fence" is a primitive that indicates whether an operation on a dma-buf has completed or not. For the camera device described above, for example, the camera driver could use a fence to signal when the buffer actually contains a new video frame. The graphics driver would then wait for the fence to signal completion before rendering the buffer to the display; it, in turn, could use a fence to signal when the rendering is complete and the buffer can be reused. Fences thus sound something like the [completion API](<https://lwn.net/Articles/23993/>), but there is additional complexity there to allow for hardware signaling, cancellation, fences depending on other fences, and more. All told, the fence patches add some 2400 lines of code to the kernel. 

The fence subsystem is meant to replace Android-specific code (called "Sync") with similar functionality. Whether that will happen remains to be seen; it [seems](<https://lwn.net/Articles/603137/>) that the Android developers have not said whether they will be able to use it, and, apparently, not all of the needed functionality is there. But there is another potential roadblock here: GPL-only exports. 

The current fence code does _not_ export its symbols with `EXPORT_SYMBOL_GPL()`; it mirrors the Sync driver (which is in the mainline staging area) in that regard. While he was reviewing the code, driver core maintainer Greg Kroah-Hartman [requested](<https://lwn.net/Articles/603139/>) that the exports be changed to GPL-only, saying that GPL-only is how the rest of the driver core has been done. That request [was not well received](<https://lwn.net/Articles/603144/>) by Rob Clark, who said: 

We already went through this debate once with dma-buf. We aren't going to change $evil_vendor's mind about non-gpl modules. The only result will be a more flugly convoluted solution (ie. use syncpt EXPORT_SYMBOL() on top of fence EXPORT_SYMBOL_GPL()) just as a workaround, with the result that no-one benefits. 

(A "syncpt" is an NVIDIA-specific equivalent to a fence). 

Greg proved to be persistent in his request, though, [claiming](<https://lwn.net/Articles/603145/>) that GPL-only exports have made the difference in bringing companies around in the past. Graphics maintainer Dave Airlie, who [came down hard](<https://lwn.net/Articles/394702/>) on proprietary graphics modules a few years ago, [disagreed](<https://lwn.net/Articles/603146/>) here, saying that the only thing that has really made the difference has been companies putting pressure on each other. Little else, he said, has been effective despite claims that some in the community might like to make. His vote was for "author's choice" in this case. 

#### Is EXPORT_SYMBOL_GPL() broken?

Dave went on to talk about the GPL-only export situation in general: 

Personally I think _GPL is broken by design, and that Linus's initial point for them has been so diluted by random lobby groups asking for every symbol to be _GPL that they are becoming effectively pointless now. I also dislike the fact that the lobby groups don't just bring violators to court. 

The last sentence above might be the most relevant in the end. For years, the kernel community has muttered threateningly about proprietary kernel modules without taking much action to change the situation. So manufacturers continue to ship such modules without much fear of any sort of reprisal. Clearly the community tolerates these modules, regardless of its (often loud) statements about the possible legal dangers that come with distributing them. 

Even circumvention of `EXPORT_SYMBOL_GPL()` limitations seems to be tolerated in the end; developers will [complain publicly](<https://plus.google.com/111049168280159033135/posts/6Zu4x37jv9G>) (sometimes) when it happens, but no further action ensues. So it should not be surprising if companies are figuring out that they need not worry too much about their binary-only modules. 

So it is not clear that `EXPORT_SYMBOL_GPL()` actually helps much at this point. It has no teeth to back it up. Instead, it could be seen as a sort of speed bump that makes life a bit more inconvenient for companies shipping binary-only modules. A GPL-only export lets developers express their feelings, and it may slow things down a bit, but, in many cases at least, these exports do not appear to be changing behavior much. The fence patches, in particular, are aimed at embedded devices, where proprietary graphics drivers are, unfortunately, still the norm. Making the interface be GPL-only is probably not going to turn that situation around. 

Perhaps one could argue that `EXPORT_SYMBOL_GPL()` is a classic example of an attempt at a technical solution to a social problem. If proprietary modules are truly a violation of the rights of kernel developers, then, sooner or later, some of those developers are going to need to take a stand to enforce those rights. The alternative is a world where binary-only kernel drivers are distributed with tacit approval from the kernel community, regardless of how many symbols are marked as being `EXPORT_SYMBOL_GPL()`. 

As with the dma-buf case, no resolution to the question of how symbols should be exported from the fence subsystem has been posted. But Greg has [said](<https://lwn.net/Articles/603187/>) that he will not give up on this particular issue, and, as the maintainer who would normally accept a patch set in this area, he is in a fairly strong position to back up his views. We may have to wait until this code is actually merged to see which position will ultimately prevail. But it seems that, increasingly, some developers will wonder if it even matters.
