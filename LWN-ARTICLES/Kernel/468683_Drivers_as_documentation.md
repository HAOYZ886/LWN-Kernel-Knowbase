---
title: Drivers as documentation
url: https://lwn.net/Articles/468683/
date: "November 22, 2011"
category: "Development model-Kernel quality"
author: "By Jonathan Corbet November 22, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 22, 2011

As a community, we are highly concerned with the quality of our code. Kernel code is reviewed for functionality, long-term maintainability, documentation, and more. Driver code is not always reviewed to the same degree, but it can be just as important - if our drivers do not work, our kernel does not work. There is an aspect to the long-term maintainability of drivers that could use more attention: the degree to which a driver documents how its hardware works. 

One might argue that the job of documenting the hardware falls on whoever writes the associated datasheet. There is some truth to that claim, but, in many cases, only the original author of the driver has access to that datasheet. Those who come after can try to extract documentation from the vendor or to search for clandestine copies hosted on the net. But often the only option is to figure out the hardware from the one source of information that is actually available: the existing driver. If the driver source does not help that new developer, one can argue that the original author has fallen down on the job. 

So, if a driver contains code like: 

```
writel(devp->regs[42], 0xf4ee0815);
```

it is missing something important. In the absence of the datasheet, there is no way for any other developer to have any clue of what that operation is actually doing. 

The problem is worse than that, though; datasheets often omit useful information, obscure the truth, and lie through their teeth. The hardest part of getting a driver to work is often the process of figuring out what the hardware's features and special needs really are. It often seems, for example, that the datasheet is written before the process of designing the hardware begins. As time passes, the understanding of the problem grows, and deadlines loom, hardware engineers start to jettison features that cannot be made to work in time or that, in their sole and not-subject-to-appeal opinion, can be painlessly fixed in software. Updating the datasheet to match the actual hardware never happens. 

Thoughtful driver developers will, on discovery of the imaginary nature of a specific hardware feature, add a comment to the driver; that way, no future maintainer has to figure out (the hard way, involving keyboard imprints on the forehead) why the driver does not use a specific, helpful-looking hardware capability. 

Then there is the matter of "reserved" bits. There has not yet been a datasheet written that did not contain entries like: 

> Weird tangential functions register (offset 0xc8)  
> ---  
> Bits| Function  
> 17| Reserved: do not touch this bit or the terrorists will win  
  
Somewhere, deep within the company, there will be a maximum of two engineers who know that the document is incomplete, but that nobody had ever gotten around to updating it. If you can corner one of those people, you can usually get them to admit that this bit should be documented as: 

> Weird tangential functions register (offset 0xc8)  
> ---  
> Bits| Function  
> 17| 0 = DMA engine randomly locks up  
>  1 = DMA engine functions as expected  
>  Default value = 0   
  
A developer who cannot get his hands within range of the neck of at least one of those hardware engineers will likely spend a lot of time figuring out that they need to set the "make it work" bit. This effort can involve reverse-engineering proprietary drivers or, in cases of pure desperation, playing with random bits to see what changes. Once that bit has been located, it is natural for the tired and frustrated developer to quietly set the bit before heading off in a determined effort to eliminate the memory of the entire process through the application of large amounts of beer. A particularly forward-thinking developer might make a note on a printed version of the datasheet for future reference. 

But handwritten notes are not usually helpful to the next developer who has to work on that driver. A moment spent documenting that bit: 

```
#define WTF_PRETTY_PLEASE  0x00020000 /* Always set this or it locks up */
```

may save somebody else hours of unnecessary pain. 

It is tempting to think of a completed driver as being done. But driver code, like other kernel code, is subject to ongoing change. Kernel API changes must be dealt with, problems need to be fixed, and newer versions of the hardware must be supported. Depending on how much beer was involved, the original author may remember that device's peculiarities, but those who follow will not. Everybody would be better served if the driver did not just make the hardware work, but if it also made the reader understand _how_ the hardware works. 

Doing so is not usually hard. Define descriptive names for registers, bits, and fields rather than putting in hard-coded constants. Note features that are incompletely described, incorrectly described, or entirely science-fictional. Comment operations that have non-obvious ordering requirements or that do not play well together. And, in general, code with a great deal of sympathy for the people who will have to make changes to your work in the future. Some hardware can never be properly documented because the relevant information is simply not available; see [this 2006 article](<https://lwn.net/Articles/203562/>) for an example. But what information is available should be made available to others. 

Core kernel hackers are occasionally heard to make dismissive remarks about driver developers and the work they do. But driver writers are often given a difficult task involving a fair amount of detective work; they get this task done and make our hardware work for us. Writing drivers that adequately document the hardware is not an unreasonable thing to ask of these developers; they have the hardware knowledge and the skills to do it. The harder problem may be asking driver _reviewers_ to insist that this extra effort be made. Without pressure from reviewers, many drivers will never enable readers to really understand what is going on.
