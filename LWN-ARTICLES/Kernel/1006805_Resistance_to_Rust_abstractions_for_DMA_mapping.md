---
title: Resistance to Rust abstractions for DMA mapping
url: https://lwn.net/Articles/1006805/
date: "January 30, 2025"
category: "Development tools-Rust; Direct memory access"
author: "By Jonathan Corbet January 30, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 30, 2025

While the path toward the ability to write device drivers in Rust has been anything but smooth, steady progress has been made and that goal is close to being achieved — for some types of drivers at least. Device drivers need to be able to set up memory areas for direct memory access (DMA) transfers, though; that means Rust drivers will need a set of abstractions to interface with the kernel's DMA-mapping subsystem. Those abstractions have run into resistance that has the potential to block progress on the Rust-for-Linux project as a whole. 

DMA transfers move data directly between RAM and the device of interest, without involving the CPU. It is difficult to get any sort of reasonable I/O performance without DMA, so almost all devices support it. Making DMA work, though, is not just a matter of handing a memory address to a peripheral device; there are many concerns that must be dealt with. These include maintaining cache coherency, ensuring that pages are resident in RAM, handling device-specific addressing limitations, programming I/O memory-management units, and more. Plus, of course, every architecture does things differently. The DMA-mapping layer exists to hide most of these problems from device drivers behind an architecture-independent interface. 

Drivers written in Rust will need to do DMA, so they will need access to the mapping layer. There have been patches to provide some of that access in circulation for some time; Abdiel Janulgue posted [a version of this work](<https://lwn.net/ml/all/20250108122825.136021-1-abdiel.janulgue@gmail.com>) in early January. This series adds a small Rust module with sufficient support to set up coherent mappings (long-term mappings in cache-coherent memory) for drivers. This work only covers part of the DMA API, but it is sufficient for simpler devices. Upcoming drivers will require that these abstractions are in place. 

But Christoph Hellwig, who does a lot of work with the DMA-mapping layer, [turned this submission away](<https://lwn.net/ml/all/20250108135951.GA18074@lst.de>) with a message reading, in its entirety: ""No rust code in kernel/dma, please"" (despite the fact that the patch did not put any code in that directory). When pressed, he [added](<https://lwn.net/ml/all/20250108151858.GB24499@lst.de>) that developers should keep these abstractions in their own code and [said](<https://lwn.net/ml/all/20250110083955.GA5395@lst.de>) that he had no interest in maintaining multi-language code. Rust developers should keep their wrapping code to themselves, he concluded. 

Danilo Krummrich [pointed out](<https://lwn.net/ml/all/Z4D5a5NYrAbNxUL6@pollux>) that the proposed abstractions were doing exactly that — keeping the Rust code separate from the rest: ""We wrote a single piece of Rust code that abstracts the C API for all Rust drivers, which we offer to maintain ourselves"". The conversation then went quiet for several days, after which Krummrich [said](<https://lwn.net/ml/all/Z4kG5AcVeQKegLnb@pollux>): ""Since there hasn't been a reply so far, I assume that we're good with maintaining the DMA Rust abstractions separately"". 

Hellwig, though, [made it clear](<https://lwn.net/ml/all/20250128092334.GA28548@lst.de>) that he is not on board with that plan. He does not want the Rust code anywhere near the DMA layer, and that fact that somebody else would be maintaining it does not change his view. Adding another language (he was clear that he was talking about any language, not Rust in particular) would, he said, make Linux as a whole ""impossible to maintain"". That has, for now, brought the conversation to a halt. 

Without DMA support, there can be no interesting drivers written in Rust. So one option that the Rust-for-Linux developers have at this point is to give up on the whole thing and find a less frustrating project to work on. As appealing as this option might be, it still is probably not their first choice, though. 

An alternative would be to do what Hellwig is suggesting and put the abstractions into each driver that needs them. That, however, is not a path toward a more maintainable kernel. When the DMA API changes, as it inevitably will, numerous drivers will have to be fixed, one by one, rather than fixing a single set of abstractions that are used by all. So this, too, might not appear at the top of the list of options as seen by the developers involved. 

Yet another approach might be to stash the DMA abstractions somewhere out of Hellwig's immediate sight — not in the `kernel/dma` directory, in other words. At that point it becomes just another user of the DMA API that, in theory, is not subject to more scrutiny than any other driver. The only problem with this idea is that Janulgue's patch already does that, and it was not sufficient. 

Someday, there will need to be a more decisive answer to this question. Krummrich has tried to bring this about with [a note](<https://lwn.net/ml/all/Z5qeoqRZKjiR1YAD@pollux>) asking for Linus Torvalds or Greg Kroah-Hartman to make a decision regarding these abstractions. Other Rust developers have [reiterated](<https://lwn.net/ml/all/293df3d54bad446e8fd527f204c6dc301354e340.camel@mailbox.org>) that they would take responsibility for the maintenance of this code, and that it would not affect the DMA subsystem. Jason Gunthorpe [questioned](<https://lwn.net/ml/all/20250130154646.GA2298732@nvidia.com>) that last claim, noting that a 6.14 pull request was delayed due to a Rust build problem, but Kroah-Hartman [answered](<https://lwn.net/ml/all/2025013030-gummy-cosmic-7927@gregkh>) that it was ""a tooling issue that people missed due to the holidays"" rather than an example of Rust code holding up development. Neither he nor Torvalds has made any decrees on whether the code in question will be merged, though. 

By allowing the entry of Rust, the kernel community has decided — on a provisional basis, at least — that it is indeed willing to maintain a multi-language code base. Perhaps, for now, the desire to banish Rust code to the periphery of the kernel makes some sense, while Rust is still seen as an ongoing experiment. If it is eventually decided that the Rust experiment has failed, backing the existing Rust code out will be easier if it's confined to the edges. 

But it seems increasingly unlikely that the Rust experiment will be judged that way. Rust clearly can be used to write kernel code, and there would appear to be some significant advantages to doing that. If the experiment has indeed succeeded then, at some point, the language will need to be treated as a first-class citizen within the kernel. Over time, "I don't want to deal with more than one language" will be an increasingly weak argument against a contribution written in Rust. 

That day may be a while in coming yet. Already overworked kernel maintainers will have to find time to learn Rust well enough to manage it within their subsystems. Incoming Rust developers can shoulder some of that burden, but they too will need time to acquire anywhere near the level of experience that the current maintainers have — experience that the kernel community depends on heavily. A change of this magnitude to a body of code as large as the kernel was never going to be a quick or easy affair; it has gone as well as could have been expected so far, but there will be more, perhaps harder, obstacles to overcome in the future.
