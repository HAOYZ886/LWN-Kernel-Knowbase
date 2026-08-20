---
title: Checkpoint/restart tries to head towards the mainline
url: https://lwn.net/Articles/320508/
date: "February 25, 2009"
category: Checkpointing; Containers
author: "By Jake Edge February 25, 2009"
---

By **Jake Edge**  
February 25, 2009

In kernel development, there is always tension between the needs of a new feature versus the needs of the kernel as a whole. Projects generally want to get their code merged as early as possible, for a variety of reasons, while the rest of the kernel community needs to be comfortable that the feature is sensible, desirable, and, perhaps most importantly, maintainable. The current push for inclusion of a feature to [checkpoint and restart](<http://lwn.net/Articles/293575/>) processes highlights this tension. 

In late January, Oren Laadan [posted](<http://lwn.net/Articles/316819/>) the latest version of his kernel-based checkpoint and restart code with the notation: ""Aiming for -mm"". There are many possible uses for checkpoints, but it is an _extremely_ complex problem. Laadan's current version is quite minimal, implementing only a fairly small subset of the features envisioned, but he would like to get the kind of review and testing that goes along with pushing it towards the mainline. 

After two weeks without much in the way of comments, another proponent, Dave Hansen [asked](<https://lwn.net/Articles/320577/>) what, if anything, was holding the patchset back from -mm inclusion. Andrew Morton [replied](<https://lwn.net/Articles/320569/>) that he had raised some concerns which were ""inconclusively waffled at"" a few months back. Morton's opinion carries a fair amount of weight--not least because he runs the targeted tree. He is looking to the future and trying to ensure that the patches make sense: 

I am concerned that this implementation is a bit of a toy, and that we don't know what a sufficiently complete implementation will look like. There is a risk that if we merge the toy we either: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

a) end up having to merge unacceptably-expensive-to-maintain code to make it a non-toy or 

b) decide not to merge the unacceptably-expensive-to-maintain code, leaving us with a toy or 

c) simply cannot work out how to implement the missing functionality. 

Morton asked for answers to several questions regarding what features are available in the current implementation, as well as information on what needs to be added. He also asked for indications that Laadan and Hansen had some thoughts on the design for required, but not yet implemented, features. In short, he wants to avoid any of the scenarios he outlined. In response to further questions from Ingo Molnar, Hansen [outlined](<https://lwn.net/Articles/320592/>) some of the shortcomings of the current implementation: 

Right now, it is good for very little. An app has to basically be either specifically designed to work, or be pretty puny in its capabilities. Any fds that are open can only be restored if a simple open();lseek(); would have been sufficient to get it back into a good state. The process must be single-threaded. Shared memory, hugetlbfs, VM_NONLINEAR are not supported. 

Hansen also had a more detailed [answer](<https://lwn.net/Articles/320593/>) to Morton's questions, which showed a lot of work still to be done. The current code only works for x86 architectures, for example, and only for basic file types, essentially just pipes and regular files. He likened the progress of checkpoint/restart to that of kernel scalability; it is a work in progress, not something that will ever be complete: 

We intend to make core kernel functionality checkpointable first. We'll move outwards from there as we (and our users) deem things important, but we'll certainly never be done. 

One of the main concerns is not that there is a lot still to be done, but that there may be lurking problems that either don't have solutions or can only be solved by very intrusive kernel changes. Matt Mackall looked at Hansen's list of additional features needing to be implemented and [summed up](<https://lwn.net/Articles/320628/>) the worries this way: 

I think the real questions is: where are the dragons hiding? Some of these are known to be hard. And some of them are critical [for] checkpointing typical applications. If you have plans or theories for implementing all of the above, then great. But this list doesn't really give any sense of whether we should be scared of what lurks behind those doors. 

There is, however, a free out-of-tree implementation of checkpoint/restart in the [OpenVZ](<http://openvz.org>) project. OpenVZ is a virtualization scheme using its own implementation of containers--different from that in more recent kernels--that supports checkpointing and migrating those containers. But it is a large patch, which Morton looked at several years ago and concluded that it would not be welcome in the mainline. Hansen [sees](<https://lwn.net/Articles/320646/>) OpenVZ as a useful example, but ""with all the input from the OpenVZ folks and at least three other projects, I bet we can come up with something better"". 

An incremental approach to implementing checkpoints is reasonable, but Morton is [concerned](<https://lwn.net/Articles/320653/>) that by merging the current patches, the kernel developers will be committed to merging something that looks a lot like--and is as intrusive as--the OpenVZ patches. Molnar is [more upbeat](<https://lwn.net/Articles/320655/>): he sees it as an important feature without ""many long-term dragons"". He does see one potential problem area in the incremental approach, though: 

There is _one_ interim runtime cost: the "can we checkpoint or not" decision that the kernel has to make while the feature is not complete. 

That, if this feature takes off, is just a short-term worry - as basically everything will be checkpointable in the long run. 

That is one of the technical issues still to be resolved with the current patchset: how does a process programmatically determine whether it is able to be checkpointed? If the process has performed some action while running on a kernel that does not support checkpointing the state caused by that action, there is a need to be able to decide that. Molnar suggested overloading the LSM security checks such that performing those actions sets a one-way "not checkpointable" flag as appropriate. That flag could be checked by the process or by some other program that was interested. Overloading the LSM hooks is not completely uncontroversial, but it does hook the kernel in many of the right places--adding an additional call to those same places for checkpointing is not likely to fly. 

There was also some question about whether the "not checkpointable" flag needs to be a one-way flag, as it could be cleared once the process has returned to a state that is able to be checkpointed. Molnar [argued](<https://lwn.net/Articles/320680/>) that the one-way flag is desirable: ""uncheckpointable functionality should be as painful as possible, to make sure it's getting fixed"". Users who run into problems checkpointing their applications will then apply pressure to get the requisite state added to checkpoints. As a starting point, Hansen has [posted](<https://lwn.net/Articles/320044/>) a patch that would add a one-way flag based on the kinds of files a process had opened. 

Checkpoints are a useful feature that could be used for migrating processes to different machines, protecting long-running processes against kernel crashes or upgrades, system hibernation, and more. It is a difficult problem that may never really be completely finished and it touches a lot of core kernel code. For these reasons, caution is certainly justified, but one gets the sense that some kind checkpoint/restart feature will eventually make its way into the mainline. Whether it is Laadan's version, something derived from OpenVZ, or some other mechanism entirely remains to be seen.
