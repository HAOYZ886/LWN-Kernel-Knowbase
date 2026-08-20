---
title: An interview with the new embedded maintainers
url: https://lwn.net/Articles/284767/
date: "June 4, 2008"
category: Embedded systems
author: "By Jake Edge June 4, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
June 4, 2008

Embedded Linux is getting a lot of attention these days. A new kernel.org mailing list, [linux-embedded](<http://vger.kernel.org/vger-lists.html#linux-embedded>)--[archived here](<http://dir.gmane.org/gmane.linux.kernel.embedded>)--has been set up, with discussions and patches already being posted. In addition, Paul Gortmaker and David Woodhouse have volunteered to be the "embedded maintainers" for the kernel to help coordinate the embedded Linux community. They graciously agreed to a joint email interview to shed some light on their new roles. 

**LWN:** What is your background with Linux, especially with embedded Linux? 

**David** : I got involved in Linux while I was at University, and ended up working at Nortel during one of the summer vacations, on a project for networking over mains power lines. It involved Linux boxes as routers, and I was working on solid state storage for that. From that, and from the basic support we had for similar devices in the PCMCIA code base, the MTD [Memory Technology Device] subsystem grew. 

After a while, I ended up working for Red Hat's engineering services division, doing board ports, drivers and other work. That's when JFFS2 was written, as part of a customer contract. 

I've been at Red Hat since 2000, in various roles including spending most of the last couple of years on OLPC. Due to HR misconduct, I handed in my notice on Monday and will be going elsewhere. I spoke to my new boss before volunteering for the 'embedded maintainer' role, and he was happy with that--it's another Linux-friendly company where I'll be doing kernel development, and community interaction will continue to be part of my day job. 

**Paul** : I started using Linux back in the pre 1.0 days, and having always been one to take things apart and see how it works, being able to do that with the OS appealed to me. I put together various documents to help people back when the entry level into Linux was quite high, started fixing and writing drivers, and on it went from there. In 2005, I joined Wind River, where I've been primarily focused on kernel and board specific kernel patches, and this has given me the opportunity to be exposed to all the different architectures and lots of board variants within each architecture family. 

**LWN:** What is the role you see for the embedded Linux maintainers for the kernel? 

**David** : A bunch of things really. It's not like a normal maintainer role where we take ownership of a certain section of code; it's a bit more fluid. 

To start with, one of the things we really need to do is work with the various people who are using Linux in "embedded" situations, and help them to work better with the community. That isn't just the vendors of consumer equipment--it's communities like OpenWRT, handhelds.org, OLPC too. In no other field is the development of the Linux kernel so balkanised, with people all over the place carrying their own patches or even full trees of code. 

Another part of the job, which is actually something I've been doing for years anyway, is reviewing general changes in the kernel with a particular mind to how they affect embedded systems. That's not just bloatwatch, although obviously that's a part of it. It also covers things like watching the IBM zSeries folks provide execute-in-place support for block devices under z/VM, and saying "hey, how can we use the same memory management for XIP from flash?". 

The other main part of it is implementing features in the core kernel which are motivated by "embedded" requirements. Like the tricks for compiling parts of the kernel with "`-fwhole-program --combine`" to let [GCC optimise better and reduce code size](<http://lwn.net/Articles/197097/>), for example. 

A certain amount of it, especially the new linux-embedded@vger.kernel.org list, I expect to be a kind of targeted kernelnewbies--but obviously with a more specific focus on embedded issues, and to a certain extent on professional developers rather than having such a high proportion of hobbyists. Although I certainly wouldn't want to discourage the hobbyists and students from getting involved with embedded. It's a good way to get people to send you cute toys, after all! 

I was trying to avoid having a 'linux-embedded' git tree, but for small things like the patch Tim Bird just sent to the linux-embedded list to introduce CONFIG_CONSOLE_TRANSLATIONS, I suppose it makes sense--so I've created that at [git://git.infradead.org/embedded-2.6.git](<http://git.infradead.org/embedded-2.6.git>). 

**Paul** : There are several things that can be done here that will all benefit Linux and its users in the end. To start with, I'm hoping that we can close some of the entry level gap between people who don't necessarily track kernel development but yet have decided to develop on Linux with a specific embedded use case in mind, and those people who are long time Linux developers. We can also improve the linkage between people writing feature changes and some of the users of those features who are likely to be impacted, but otherwise would probably go unheard from. We can also look at externally maintained features of interest to embedded users, and try and determine what is the blocking factor that is stopping it (or parts of it) from being merged upstream, and then assist in removing those barriers where possible. 

**LWN:** What are the specific problems that are faced by embedded developers trying to use Linux? What can you do to make that situation better? 

**David** : I think the biggest single problem has always been the same--it's that people are too focused on getting their stuff out the door as quickly as possible without much thought to working with upstream. Managers aren't budgeting the time to get things merged, and engineers aren't talking about their design early enough that it can be improved before it's a fait accompli. That extra time isn't just about being a good citizen--failing to do it almost _always_ comes back to bite you personally, when you come to do a new product, a product update, or even need to merge in changes from upstream to fix bugs. But _everybody_ seems to need to learn that the hard way, it seems. 

**Paul** : A lot of times, you get the situation where a group who is developing for an embedded platform is focused 100% on getting their product up, running and deployed. The developers involved aren't necessarily hard core Linux folks, and it usually plays out by them picking a kernel version, getting their stuff in their local tree, and that is it. They may not know git, they probably don't have insight into who the respective subsystem maintainers are, they may perceive LKML as too hostile, or they may not have management buy-in on trying to push stuff upstream. But inevitably, some time passes, and then they have a carry forward task where they try and do a big jump uprev of all their changes, and this repeats forever. 

Most people who have had to endure the jump uprev vs. a continual tracking and carrying of changes will tell you the jump is not the way to go for a multitude of reasons, but it seems a lesson that everybody ends up having to learn on their own. So, I'm hoping we can get some of these people more aligned with the typical Linux developer workflow--i.e. work from the latest codebase, create logical changesets that can be submission candidates etc. I've been in a couple of meetings recently where we've had the opportunity to educate embedded developers on the advantages of doing this, and the feedback has been positive so far. 

**LWN:** The size of the kernel is getting larger in general, is it getting too big for some embedded applications? What, if anything, should be done to remedy that situation? 

**David** : I know there are people who'll want to take me out back and shoot me for this, but I think a large part of the solution to that is knowing when Linux is the answer, and accepting that sometimes it isn't. I've always been a bit dubious about implementing XIP support in Linux, for example, on the basis that if you care _that_ much, you should probably have been using something like eCos anyway. 

Getting back to the real question, though, there _are_ things we can do. The smaller, more efficient "slub" memory allocator is an example, as is the `--combine` thing I mentioned above. The trick is to find ways to improve matters without just littering the whole thing with ifdefs. 

**Paul** : There will always be some hardware or some use case where Linux isn't the right choice. It only makes sense to use the right tool for the job. However we do want to make sure that Linux is that right tool in as many cases as possible. On the plus side, the resources that are found on a typical embedded target today are a lot more rich than they were years ago. We just need to make sure that in optimizing for the general x86 use case, we don't inadvertently hinder these more fringe use cases coming from the embedded world. 

**LWN:** What do you see as the priorities for kernel work to better support embedded Linux? 

**David** : One important priority right now is replacing JFFS2. I wrote it, so I'm allowed to say that--it was good for its time, with NOR flash devices on the order of 32MiB. But having made it work on 1GiB of NAND flash in OLPC, I certainly agree with the observation that it's being pushed past its design limits. I'm very keen to get LogFS and/or UBIFS merged into the kernel and stabilised to the point where we can really start moving to them. 

We need to revamp the MTD API fairly urgently too. It was derived from the PCMCIA code we had at the time without much planning, and we really need to improve on it now. 

There may be a certain amount of bias in the items I've picked out, I suppose. 

**Paul** : The embedded community as a whole is probably the biggest user of all the architectures outside of the x86 based platforms. Sometimes the functionality of certain things don't get much testing outside of the basic x86 family. For example, one of the features that there is considerable interest in is the full preempt_rt patch set. Yet once you stray outside of the x86 family, you are pretty much guaranteed to run into drivers specific to embedded targets that don't play nice once this patch set is in place. This isn't such a surprise, simply because the intersection of the two hasn't been explored yet. I think there is value here in getting these types of intersections explored sooner rather than later, by reducing some of the gap between the people working on these sorts of features, and those intending to use them on embedded platforms. 

**LWN:** Do you have any specific goals for timelines of getting various features merged? 

**David** : Other than "ASAP" for LogFS and UBIFS, not particularly. Stuff is merged when it's ready. 

**Paul** : At this point in time, no. I'm not really interested in hijacking anyone's project or feature and trying to drive it towards some self-imposed merge deadline. I'd rather work with them to try and find out what the problem areas are, help with those where possible, be they logistical or technical and get them to a point where they feel that they can offer up merge candidates. 

**LWN:** What problems do you foresee in working with other kernel developers who may have less (or no) interest in the concerns of the embedded community? Are there specific features that may be difficult or impossible to get merged? 

**David** : I know it's fashionable to claim there's a big disconnect between embedded and big-iron users, but actually there's a lot more overlap than many people seem to realise. I mentioned XIP earlier; can you also guess who was first to implement tickless support? 

A lot of the problem has been people who show up and throw their code over the wall, then run away. Or worse, those who don't even throw it over the wall at all. People seem to have forgotten how long it took us to educate the enterprise vendors and get them to work nicely with us; we're a bit behind the curve on the embedded side but we're getting there. And organisations like CELF are doing good work on that front, too. 

**Paul** : We have to be realistic. There will always be some features that either are too invasive to be sensible merge candidates, or the particular feature has such a small user base, that it may not make sense from a carrying cost point of view to target it for inclusion in the standard kernel. Fortunately, I think the Linux developer community at large has generally been flexible in accommodating most things, while at the same time excluding things where the best interest of the kernel as a whole needed to come first. 

In such cases where a feature doesn't look to be a probable merge candidate, not all is lost. We have to capitalize on the remaining value adds that come with still working with it as if it was a merge candidate. Things like cherry-picking parts of it that are of global value and thus reducing the carrying cost. Or being able to voice an opinion at the appropriate time if the maintainer of the feature notices that a proposed change somewhere else in the kernel will impact the feature that they have been maintaining independently. So I think we still want to work towards getting the people handling these "harder" features of interest to the embedded community working more in parallel with the main kernel community. 

**LWN:** The term "embedded Linux" covers a huge spectrum of devices and uses of Linux, everything from devices where the OS is completely invisible up through internet tablets and UMPC devices that are essentially desktops squeezed into a smaller package. Where on that spectrum do your interests lie? What do you think the challenges of trying to support all of those different uses will be? 

**David** : My interest is everywhere in that spectrum--and beyond. Too much focus on one small area is the way to ensure that you solve your own problems while pessimising things for other people. I think it's important to keep a certain amount of holistic focus, because that's how we can make sure that Linux scales well both up and down. 

**Paul** : Absolutely. It seems that people naturally associate embedded with the small and resource constrained end of the scale. But the reality is that there are people who are wanting to use Linux in embedded applications where the baseline hardware has 16 cores and gigabytes of memory. On the one end of the scale you are interested in things like efficiency of resource usage, quick boot times, and on the other end of the scale, your interests are more likely around features relating to specific high availability features that may not be present in the standard kernel tree. 

These are clearly separate problem spaces, but the common thing they both share is that you've got a group using a specific piece of hardware with a specific use case in mind. This tends to bring out the "works for us, lets get it done and shipping" mentality, and the work tends to never make it out to where others can review it and look at merging bits that make sense. I'm hoping this is where we can make a difference. 

We would like to thank David and Paul for taking time to answer these questions.
