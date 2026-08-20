---
title: On the management of the Video4Linux subsystem tree
url: https://lwn.net/Articles/320472/
date: "February 24, 2009"
category: Development model; Video4Linux2
author: "By Jonathan Corbet February 24, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
February 24, 2009

Once upon a time, the Video4Linux (V4L) development community was seen as a discordant group which hung out in its own playpen and which had not managed to implement support for much of the available hardware. Times have changed; the V4L community is energetic and productive, disruptive flame wars have all but disappeared from the V4L mailing lists, and Linux now supports a large majority of the hardware which can be found on the market. As this community moves forward, it is reorganizing things on many fronts; among other things, they are working on the creation of the first true framework for video capture devices. The V4L developers are also having to look at their code management practices; in the process they are encountering a number of issues which have been faced by other subsystems as well. 

The discussion started with [this RFC from Hans Verkuil](<https://lwn.net/Articles/320473/>). Hans points out that the size of the V4L subsystem (as found under `drivers/media` in the kernel source) has grown significantly in recent years - it is 2.5 times larger now than it was in the 2.6.16 kernel. This growth is a sign of success: V4L has added features and support for a vast array of new hardware in this time. But it has its costs as well - that is a lot of code to maintain. 

As it happens, the V4L developers make that maintenance even harder by incorporating backward compatibility into their tree. The tree run by V4L maintainer Mauro Carvalho Chehab does not support just the current mainline kernel; instead, it can be built on any kernel from 2.6.16 forward. This is not a small trick, considering that the majority of that code did not exist when 2.6.16 was released. There have been some major internal kernel API changes over that time; supporting all those kernels requires a complicated array of `#ifdef`s, compatibility headers, and more. It takes a lot of work to keep this compatibility structure in place. Additionally, this kind of compatibility code is not welcome in the mainline kernel, so it must all be stripped out prior to sending code upstream. 

The reason for this practice is relatively straightforward: the V4L developers would like to make it possible for testers to try out new drivers without forcing them to install a leading-edge mainline kernel. This is the same reasoning that the DRM developers gave at the [2008 Kernel Summit](<http://lwn.net/Articles/298842/>): allowing testers to build modules for older kernels makes life easier for them. And that, in turn, leads to more testing of current code. But the cost of this compatibility is high, so Hans is proposing a few changes. 

One of those would be in how the subsystem tree is managed. Currently, this tree is maintained in a Mercurial repository which represents only the V4L subsystem (it is not a full kernel tree), and which contains the backward compatibility patches. This organization makes interaction with the kernel development process harder in a number of ways. Beyond the effort required to maintain backward compatibility, the separate tree makes it harder to integrate patches written against the mainline kernel, and there is no way for this tree to contain patches which affect kernel code outside of `drivers/media`. Life would be easier if developers could simply work against an ordinary mainline kernel tree. 

So Hans suggests moving to a tree organization modeled on the techniques developed by the ALSA project. The ALSA maintainers (who also keep backward compatibility patches) use as their primary tree a clone of the mainline git repository. Backward compatibility changes are then retrofitted into a separate tree which exists just for that purpose. By working against a mainline tree, the ALSA developers interact more smoothly with the rest of the kernel development process. The down side is that creating the backward-compatible tree requires more work; a team of V4L developers would have to commit to putting time toward that goal. 

And that leads, of course, to the biggest question: what is the real value of the backward compatibility work, and how far back should the project go? There seems to be little interest in dropping compatibility with older kernels altogether; the value to testers and developers both seems to be too high. But it is not clear that it is really necessary to support kernels all the way back to 2.6.16. So, asks Hans, what is the oldest kernel that the project should support? 

Hans has a clear objective here: the i2c changes which were merged for 2.6.22 create a boundary beyond which backward compatibility gets significantly harder. If kernels before 2.6.22 could be dropped, a lot of backward compatibility hassles would go away. But convenience is not the only thing to bear in mind when dropping support; one must also consider whether that change will significantly reduce the number of testers who can try out the code. It would also be good to have some sort of objective policy on backward compatibility support so that older kernels could be dropped in the future without the need for extensive discussions. 

The proposed policy is this: V4L backward compatibility should support the oldest kernels supported by "the three major distros" (Fedora, openSUSE, and Ubuntu). For the moment, that kernel, conveniently, happens to be 2.6.22, which will be supported by Ubuntu 7.10 until April, 2009. (Interestingly, Hans seems to have skipped over the 6.06 "Dapper Drake" release - supported until June, 2009 - which runs a bleeding-edge 2.6.15 kernel). A quick [poll](<https://lwn.net/Articles/320489/>) run by Hans suggests that there is little opposition to removing support for kernels prior to 2.6.22. 

There is some, though: John Pilkington [points out](<https://lwn.net/Articles/320490/>): 

I think you should be aware that the mythtv and ATrpms communities include a significant number of people who have chosen to use the CentOS_5 series in the hope of getting systems that do not need to be reinstalled every few months. I hope you won't disappoint them. 

CentOS 5 (like the RHEL5 distribution it is built from) shipped with a 2.6.18 kernel. It seems, though, that [there is little sympathy for CentOS](<https://lwn.net/Articles/320491/>) (or any other "enterprise" distribution) in the development community. Running a distribution designed to be held stable for several years and wanting the latest hardware support are seen to be contradictory goals. So it seems unlikely that the V4L tree will be managed with the needs of enterprise distributions in mind. 

Thus far, no actual decisions have been made. Mauro, who as the subsystem maintainer would be expected to have a strong voice in any such decision, has not yet shown up in the discussion. Given the lack of any strong opposition to the proposals, though, it would be surprising if those proposals are not adopted in some form.
