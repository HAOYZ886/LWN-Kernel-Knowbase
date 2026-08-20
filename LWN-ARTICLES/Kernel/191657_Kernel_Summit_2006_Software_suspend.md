---
title: "Kernel Summit 2006: Software suspend"
url: https://lwn.net/Articles/191657/
date: "July 18, 2006"
category: Software suspend
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
Pavel Machek got up to talk about the current state of software suspend in the Linux kernel. He started with a long set of instructions on how to make suspend-to-disk work on a Linux system; happily for most of us, a growing number of Linux distributors are taking care of making it work so we don't have to deal with all that. Additionally, he claimed, suspend-to-RAM works on most machines - interesting news to your editor, whose laptop has a distressing tendency to corrupt its disk when suspended to RAM. 

What people really wanted to talk about, however, is the user-space suspend-to-disk interface. According to Pavel, the in-kernel suspend implementation is deprecated, and will eventually be removed in favor of the user-space version. This approach makes it easier to add features like encryption and compression, and a nicer user interface as well. 

A fair number of kernel developers are, however, unconvinced that the user-space approach makes sense. Encryption and compression can be done within the kernel, using the infrastructure which already exists there. The suspend2 patches by Nigel Cunningham show that a nice interface can be implemented in user space, even though few seem to think that suspend2 is the right way to do it. In general, what the kernel developers - and their long-suffering users - would like to see is a suspend implementation that simply _works_ ; progress bars are something which can be worried about later on. The current user-space work, says Andrew Morton, is "madness." 

It was pointed out that the suspend2 patches have a lot of happy users. This implementation is said to be more robust. A couple of reasons for this perception were floated. One is that suspend2 is generally the last resort of people who cannot get the in-kernel software suspend to work; those who have success with suspend2 will, naturally, see it as being better. Ted Ts'o noted that suspend2 has an active user and developer community which provides a high level of support to people trying to make it work. Users of in-kernel suspend, instead, tend to be on their own. The existence of this support system makes a huge difference for people trying to set up a tricky feature like suspend-to-disk. 

That said, there is still little interest in bringing the suspend2 code into the kernel. The quality of this code was criticized - and Nigel was not there to defend it. Whether or not those criticisms are valid, it is true that the suspend2 patches are huge, and that Nigel has not been particularly effective in his dealings with the rest of the kernel development community. Getting any sizeable portion of suspend2 merged seems like an unlikely prospect. 

In any case, the real problem with software suspend is not the core code, which is highly similar between all of the implementations. It lies, instead, with device support. In many cases, the required fixes are said to be relatively straightforward, but people are not doing that work. Until that changes, software suspend is likely to remain a tricky affair. 

  * [Next: Documentation](<https://lwn.net/Articles/191659/>).
