---
title: "LPC: Coping with hardware diversity"
url: https://lwn.net/Articles/458729/
date: "September 14, 2011"
category: Architectures
author: "By Jonathan Corbet September 14, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 14, 2011

As Linaro's CTO, David Rusling spends a lot of time observing the interactions between the ARM architecture and the mainline kernel development community. In his Linux Plumbers Conference 2011 keynote, David made the point that ARM's diversity is behind many of the problems that have made themselves felt in recent years. Much is being done to align the ARM community with how the kernel works, but the kernel, too, is going to have to change if it will successfully address the challenges posed by increasingly diverse hardware. 

David started with a brief note to the effect that he dislikes the "embedded" term. If a system is connected to the Internet, he said, it is no longer embedded. Now that everything is so connected, it is time to stop using that term, and time to stop having separate conferences for embedded developers. It's all just Linux now. 

#### ARM brings diversity

ARM is a relative newcomer to the industry, having been born in 1990 as part of a joint venture between Acorn, VLSI, and Apple. The innovative aspect to ARM was its licensing model; rather than being a processor produced by a single manufacturer, ARM is a processor design that is licensed to many manufacturers. The overall architecture for systems built around [![\[David Rusling\]](https://static.lwn.net/images/conf/2011/lpc/DavidRusling1a-sm.jpg)](<https://lwn.net/Articles/458770/>) ARM is not constrained by that license, so each vendor creates its own platform to meet its particular needs. The result has been a lot of creativity and variety in the hardware marketplace, and a great deal of commercial success. David estimated that each attendee in the room was carrying about ten ARM processors; they show up in phones (several of them, not just "the" processor), in disk controllers, in network interfaces, etc. 

Since each vendor can create a new platform (or more than one), there is no single view of what makes an ARM processor. Developers working with ARM usually work with a single vendor's platform and tend not to look beyond that platform. They are also working under incredibly tight deadlines; four months from product conception to availability on the shelves is not uncommon. There is a lot of naivety about open source software, its processes, and the licensing. In this setting, David said, fragmentation was inevitable. Linaro has been formed in response in an attempt to help the ARM community work better with the kernel development community; its prime mission is to bring about some consolidation in the ARM code base. Beyond that, he said, Linaro seeks to promote collaboration; without that, the community will be able to achieve very little. Companies working in the ARM space recognize the need to collaborate, but they are sometimes less clear on just which problems they should be trying to solve. 

Once upon a time, Microsoft was the dominant empire and Linux was the upstart rebel child. Needless to say, Linux has been successful in many areas; it is now settling, he said, into a comfortable middle age. But this has all happened in the context of the PC architecture, which is not particularly diverse, so Linux, too, is not hugely diverse. It's also worth noting that, in this environment, hardware does not ship until Windows runs on it; making Linux work is often something that comes afterward. 

The mobile world is different; Android, he said, has become the de facto standard mobile Linux distribution. It has become known for its "fork, rebase, repeat" development cycle. Android runs on systems with highly integrated graphics and media processors, and it is developed with an obsession about battery lifetime. In this world, things have turned around: now the hardware will not ship until Linux runs on it. Given the time pressures involved, it is no wonder, he said, that forking happens. 

In the near future we are going to see the arrival of ARM-based server systems; that is going to stir things up again. They will be very different from existing servers - and from each other; the diversity of the ARM world will be seen again. There will be a significant long-term impact on the kernel as a result. For example, scheduling will have to become much more aware of power management and thermal management issues. Low power use will always be a concern, even in the server environment. 

#### Problems to solve

Making all of this work is going to require greater collaboration between the ARM and kernel communities. ARM developers are developing the habits needed to work with upstream; the situation is much better than it was a few years ago. But we are going to need a lot more kernel developers with an ARM background, and they are going to have to get together and talk to each other more often. Some of that is beginning to happen; Linaro is trying to help with this process. 

A big problem to deal with, he said, was boot architecture: what happens on the system before the kernel runs. Regardless of architecture, the boot systems are all broken and all secret; developers hate them. In the end we have to communicate system information to the kernel; now we are using features like ACPI or techniques like flattened device trees. We are seeing new standards (like UEFI) emerging, but, he asked, are we influencing those standards enough? 

Taking things further: will there be a single ARM platform such that one kernel can run on any system? The answer was "maybe," but, if so, it is going to take some time. We're currently in a world where we have many such platforms - OMAP, iMX, etc. - and pulling them together will be hard. We need to teach ARM developers that not all code they develop belongs in their platform tree - or in `arch/arm` at all. The process of looking for patterns and turning them into generic code must continue. The ARM community is working toward the goal of creating a generic kernel; there are lots of interesting challenges to face, but other architectures have faced them before. 

One step in the right direction is the recent creation of the arm-soc tree, managed by Arnd Bergmann. The goal of this tree is to support Russell King (the top-level ARM maintainer) and the platform maintainers and to increase the efficiency of the whole process. The arm-soc tree has become the path for much of the ARM consolidation work to get into the mainline kernel. 

Returning briefly to power management, David noted that ARM-based systems usually have no fans. The kernel needs a better thermal management framework to keep the whole thing from melting. And that framework will [![\[David Rusling\]](https://static.lwn.net/images/conf/2011/lpc/DavidRusling3a-sm.jpg)](<https://lwn.net/Articles/458771/>) have to reach throughout the kernel; the scheduler may, for example, need to move processes away from an overheating core to allow it to cool down. Everywhere we look, he said, we need better instrumentation so we have a better idea of what is happening with the hardware. 

More efficient buffer management is a high priority for ARM devices; copying data uses power and generates heat, so copying needs to be avoided whenever possible. But existing kernel mechanisms are not always a good match to the ARM world, where one can encounter a plethora of memory management units, weakly-ordered memory, and more. There are a lot of solutions in the works, including [CMA](<https://lwn.net/Articles/450286/>), a [reworked DMA mapping framework](<https://lwn.net/Articles/448470/>), and more, but they are not all yet upstream. 

In summary, we have some problems to solve. There is an inevitable tension between product release plans and kernel engineering. Product release cycles have no space for the "argument time" required to get features into the mainline kernel. It is, he said, a social engineering problem that we have to solve. It will certainly involve forking the kernel at times; the important part is joining back with the mainline afterward. And, he asked, do we really need to have everything in the kernel? Perhaps, in the case of "throwaway devices" with short product lives, we don't really need to have all that code upstream. 

If we are going to scale the kernel across the diversity of contemporary hardware, he said, we will have to maintain a strong focus on making our code work on all systems. We'll have to continue to address the tensions between mobile and server Linux, and we'll have to make efforts to cross the kernel/user-space border and solve problems on both sides. This is a discussion we will be having for some time, he said; events like the Linux Plumbers Conference are the ideal place for that discussion.
