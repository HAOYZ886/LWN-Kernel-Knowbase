---
title: The BPF panic function
url: https://lwn.net/Articles/901284/
date: "July 18, 2022"
category: "BPF; Releases-6.1"
author: "By Jonathan Corbet July 18, 2022"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 18, 2022

One of the key selling points of the BPF subsystem is that loading a BPF program is safe: the BPF verifier ensures that the program cannot hurt the kernel before allowing the load to occur. That guarantee is perhaps losing some of its force as more capabilities are made available to BPF programs but, even so, it may be a bit surprising to see [this proposal from Artem Savkov](<https://lwn.net/ml/linux-kernel/20220711083220.2175036-1-asavkov@redhat.com/>) adding a BPF helper that is explicitly designed to crash the system. If this patch set is merged in something resembling its current form, it will be the harbinger of a new era where BPF programs are, in some situations at least, allowed to be overtly destructive. 

As Savkov notes, one of the major use cases for BPF is kernel debugging, a task which is also often helped by the existence of a well-timed crash dump. By making the kernel's [`panic()`](<https://elixir.bootlin.com/linux/v5.18.11/source/kernel/panic.c#L178>) function available to BPF programs, Savkov is trying to combine the two by allowing a BPF program to cause a crash — and create a crash dump — when it detects the conditions that indicate a problem that a developer is looking for. Savkov is seemingly not the only one wanting this capability; Jiri Olsa [noted](<https://lwn.net/ml/linux-kernel/YswAqrJrMKIZPpcz@krava/>) that he has gotten a request for this feature as well. 

Making `panic()` available to BPF has some obvious hazards, so one would expect that there would be some guard rails put into place. In this case, the first step is a new flag, `BPF_F_DESTRUCTIVE`, that must be provided when a program that will invoke destructive operations (such as a `panic()` call) is loaded. If this flag is not present, the BPF verifier will reject the loading of a program that contains calls to any destructive helper functions, of which `panic()` is the only one (so far). 

Even then, the `panic()` helper function is only available to tracing programs. It makes little sense, after all, for an [infrared decoder](<https://lwn.net/Articles/759188/>) to be able to panic the system, though this restriction will prevent a complete implementation in BPF for remote controls featuring a "panic" button. Then, there is a new sysctl knob (`kernel.destructive_bpf_enabled`) that must be set to a non-zero value; otherwise the `panic()` call will not be allowed. Even when the sysctl knob has been set, the process on whose behalf the BPF program is running must have the `CAP_SYS_BOOT` capability. 

All told, it seems unlikely that a BPF program will panic the system by mistake. 

There does not appear to be much in the way of opposition to this patch, though there were some questions about the details. For example, Song Liu [disliked the sysctl knob](<https://lwn.net/ml/linux-kernel/CAPhsuW7xTRpLf1kyj5ejH0fV_aHCMQjUwn-uhWeNytXedh4+TQ@mail.gmail.com/>) ""as it is global, and the user can easily forget to turn it back off"". Alexei Starovoitov also [said](<https://lwn.net/ml/linux-kernel/CAADnVQ+ju04JAqyEbA_7oVj9uBAuL-fUP1FBr_OTygGf915RfQ@mail.gmail.com/>) that a sysctl is not called for in this situation; the `CAP_SYS_BOOT` check should be enough, he said. Starovoitov also questioned the need for a full panic of the system, given that there are more direct ways to create a crash dump. Savkov [replied](<https://lwn.net/ml/linux-kernel/Ys7JL9Ih3546Eynf@wtfbox.lan/>) that `panic()` is ""more versatile"" and that the system's response to a panic is configurable by the administrator. He did agree to remove the sysctl knob. 

Starovoitov also suggested implementing the functionality as [a kfunc](<https://lwn.net/Articles/856005/>) rather than as a BPF helper. The reasoning here is that kfuncs are deemed to be unstable and can be removed at any time if they turn out to be a bad idea, while removal of BPF helpers is harder. It is worth noting, of course, that this position on the removability of kfuncs has, thus far, not faced the test of an irate user whose application depends on a kfunc that has just been removed. 

In [a later response](<https://lwn.net/ml/linux-kernel/CAADnVQ+6aN5nMwaTjoa9ddnT6rakgwb9oPhtdWSsgyaHP8kZ6Q@mail.gmail.com/>), Starovoitov questioned the "versatility" of `panic()`, and said that lower-level functions should be provided to BPF programs instead. Thus, there should be one to create a crash dump, one to send a message to the console, one to halt the system, one to reboot, and so on. That way, he said, the program itself can decide what should happen rather than depending on a specific configuration of the system. 

There is clearly another revision of this patch set coming in the future, and it may look significantly different than it did the first time around. But it also seems clear that there is a use case for this sort of "destructive" functionality in BPF programs. The BPF system is quickly growing beyond packet handling and information gathering and heading toward a point where arbitrary types of kernel functionality are available to BPF programs. It's not clear where all of this leads, but it seems likely to be interesting.
