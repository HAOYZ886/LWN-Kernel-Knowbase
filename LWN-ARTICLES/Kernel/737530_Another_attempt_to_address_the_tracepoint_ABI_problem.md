---
title: Another attempt to address the tracepoint ABI problem
url: https://lwn.net/Articles/737530/
date: "October 27, 2017"
category: "Tracing-ABI issues"
author: "By Jonathan Corbet October 27, 2017 2017 Kernel Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 27, 2017

* * *

[2017 Kernel Summit](<https://lwn.net/Articles/KernelSummit2017/>)

Tracepoints provide a great deal of visibility into the inner workings of the kernel, which is both a blessing and a curse. The advantages of knowing what the kernel is doing are obvious; the disadvantage is that tracepoints risk becoming a part of the kernel's ABI if applications start to depend on them. The need to maintain tracepoints could impede the ongoing development of the kernel. Ways of avoiding this problem have been discussed for years; at the 2017 Kernel Summit, Steve Rostedt talked about yet another scheme. 

The risk of creating a new ABI has made some maintainers reluctant to add instrumentation to their parts of the kernel, he said. They might be willing to add new interfaces to provide access to specific information but, in the absence of tools that use this information, it is hard to figure out which information is needed or what a proper interface would be. The solution might be to adopt an approach that is similar to the staging tree, where not-ready-for-prime-time drivers can go until they are brought up to the necessary level of quality. 

People talk about "tracepoints", but there are actually two mechanisms in the kernel. Internally, a tracepoint is a simple marker in the code, a hook to which a kernel function can be attached. What user space sees as a [![\[Steve Rostedt\]](https://static.lwn.net/images/conf/2017/osse/SteveRostedt-sm.jpg)](<https://lwn.net/Articles/737535/>) tracepoint is actually a "trace event", which is a specific interface that is implemented using the internal tracepoints. Without trace events, there is no interface visible to user space. 

The proposed solution to the ABI problem is to place a tracepoint at locations of interest, but not bother with the trace event. Making the tracepoint available to user space would then require loading a kernel module; this module would be kept out of the mainline tree. It would be, he said, a development space to try out interfaces for the more sensitive tracepoints. Since it is not a part of the mainline kernel, it could not become part of the kernel ABI. But distributors could ship this module, making the tracepoints available to user-space developers. 

Ben Hutchings, a Debian kernel maintainer, said that this approach would not work in a number of cases. There are many situations where it's not possible to just load a random module into the kernel. Many customers are using module signing, for example, to prevent exactly that from happening. Even if distributions ship this module, users of different distribution would have different modules and the tracepoints would be incompatible; that would make it harder to write tools to use them. 

Another member of the audience expressed skepticism, saying that if every distributor ships this module, it will become an ABI that has to be maintained anyway. Ben Herrenschmidt agreed and suggested that the right solution was to make the tracepoints be self-describing. But, as Rostedt pointed out, they are already self-describing, but changing the availability of information will still break things. Tools may depend on specific information that is no longer available, or they may simply ignore the format information for the tracepoint. That makes it hard to remove obsolete tracepoints which, since they each occupy about 5KB of memory, is unfortunate. 

Matthew Wilcox asked whether the proposed scheme would have solved the problem with powertop, which broke some years ago when a variable was removed from a tracepoint. Rostedt said that it would have; Ted Ts'o noted that the powertop problem shows that self-describing formats do not work as a solution to this problem. 

Much of the current work is being pushed by developers within Facebook, who use a vast library of tracepoints to diagnose performance problems. They are willing to deal with their tools breaking when the kernel changes. That led Andrew Morton to ask whether Linus Torvalds made the right call by including tracepoints in the kernel ABI. Rostedt said he disagrees with that decision, but it doesn't matter, since Torvalds has the final say. David Woodhouse complained that the group was talking about "arbitrary technical nonsense"; perhaps the loaded module should just set a flag to make the tracepoints available. Morton agreed that the module idea "sounds like bullshit" and suggested that perhaps it was time to get the rule changed. But Rostedt has tried that before, he said, and he still bears the scars that resulted. 

Chris Mason said that, while Facebook can handle tracepoint changes that break its tools, there is a need to know when such a change has happened. Just moving the ABI to a loadable module will not solve that problem; it just pushes the problem onto the distributors instead. 

Ts'o then launched into a discussion of the growing set of tools that work by attaching BPF scripts to tracepoints. These tools are becoming popular and soon will be as popular as powertop; that will result in the same kinds of problems when they break. The problem is here now and needs to be addressed. 

Doing so will be hard, he said. The topic had been suggested for the invitation-only Maintainers Summit, since it is "fundamentally a Linus problem", but Torvalds had [vetoed it](<https://lwn.net/Articles/737532/>). Torvalds wants to make a guarantee to user-space tools that works in 99% of the cases, but it is hard to live up to for tools that are closely tied to the kernel. So the powertop problem will come again, only worse; BPF will "turn it into a trainwreck". Rostedt added that Linux started off as "a desktop toy", but it is no longer a simple system. Nobody knows the whole thing, so they are relying more on tooling to know what is going on. 

The conversation came to an end about here, but the topic did return at the Maintainers Summit later that week, after Torvalds and Rostedt had discussed it. The solution that was arrived at for now, as related by Torvalds, is to hold off on adding explicit tracepoints to the kernel. Instead, support will be added to make it easy for an application to attach a BPF script to any function in the kernel, with access to that function's arguments. That should give tools access to the information they need, and may make it possible to (eventually) remove many of the existing explicit tracepoints. 

Arnd Bergmann asked what would happen if a popular script breaks as the result of the removal of a function; Torvalds replied that he would not see it as a regression that needs to be fixed. But, he said, if that happens it should be seen as a sign that the kernel should be providing that information in a more straightforward manner. A tracepoint or other interface could be added at that time. 

Whether this solution provides what the tools need will take time to determine. But if it does, it may just be possible that a multi-year debate has finally come to some sort of conclusion that all of the parties involved can live with. 

[Your editor would like to thank the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event].
