---
title: What to do in response to a kernel warning
url: https://lwn.net/Articles/876209/
date: "November 18, 2021"
category: Warnings
author: "By Jonathan Corbet November 18, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 18, 2021

The kernel provides a number of macros internally to allow code to generate warnings when something goes wrong. It does not, however, provide a lot of guidance regarding what should happen when a warning is issued. Alexander Popov recently posted [a patch series](<https://lwn.net/ml/linux-kernel/20211027233215.306111-1-alex.popov@linux.com/>) adding an option for the system's response to warnings; that series seems unlikely to be applied in anything close to its current form, but it did succeed in provoking a discussion on how warnings should be handled. 

Warnings are emitted with macros like `WARN()` and `WARN_ON_ONCE()`. By default, the warning text is emitted to the kernel log and execution continues as if the warning had not happened. There is a sysctl knob (`kernel/panic_on_warn`) that will, instead, cause the system to panic whenever a warning is issued, but there is a lack of options for system administrators between ignoring the problem and bringing the system to a complete halt. 

Popov's patch set adds another option in the form of the `kernel/pkill_on_warn` knob. If set to a non-zero value, this parameter instructs the kernel to kill all threads of whatever process is running whenever a warning happens. This behavior increases the safety and security of the system over doing nothing, Popov said, while not being as disruptive as killing the system outright. It may kill processes trying to exploit the system and, in general, prevent a process from running in a context where something is known to have gone wrong. 

There were a few objections to this option, starting with Linus Torvalds, who [pointed out](<https://lwn.net/ml/linux-kernel/CAHk-=wg+UMNYrR59Z31MhxMzdUEiZMQ1RF9jQvAb6HGBO5EyEA@mail.gmail.com/>) that the process that is running when a warning is issued may not have anything to do with the warning itself. The problem could have happened in an interrupt handler, for example, or in a number of other contexts. ""Sending a signal to a random process is just voodoo programming, and as likely to cause other very odd failures as anything else"", he said. 

Torvalds suggested that a better approach might be to create a new `/proc` file that will provide information when a system-tainting event (such as a warning) happens. A user-space daemon could poll that file, read the relevant information when a warning is issued, then set about killing processes itself if that seems like the right thing to do. Marco Elver [added](<https://lwn.net/ml/linux-kernel/YZEbYmzy64uai7Af@elver.google.com/>) that there is a tracepoint that could provide the relevant information with just a bit of work. Kees Cook [threw together an implementation](<https://lwn.net/ml/linux-kernel/202111151116.933184F716@keescook/>), but Popov [didn't like it](<https://lwn.net/ml/linux-kernel/59534db5-b251-c0c8-791f-58aca5c00a2b@linux.com/>); that approach would allow a process to continue executing after the warning happens, he said, and by the time user space gets around to doing something about the situation, it may be too late. 

James Bottomley [argued](<https://lwn.net/ml/linux-kernel/88093da62a4b85f015423cbd1ec2f5ad6eb0c5da.camel@HansenPartnership.com/>) that all of the approaches discussed so far were incorrect. If a warning happens, he said, the kernel is no longer in a known state, and anything could happen: 

> What WARN means is that an unexpected condition occurred which means the kernel itself is in an unknown state. You can't recover from that by killing and restarting random stuff, you have to reinitialize to a known state (i.e. reset the system). Some of the reason we do WARN instead of BUG is that we believe the state contamination is limited and if you're careful the system can continue in a degraded state if the user wants to accept the risk. 

Thus, he said, the only rational policies are to continue (accepting the risk that bad things may happen) or kill the system and start over — the options that the kernel provides now. 

Popov had [suggested](<https://lwn.net/ml/linux-kernel/77b79f0c-48f2-16dd-1d00-22f3a1b1f5a6@linux.com/>) that the [ELISA project](<https://elisa.tech/>), which is working toward Linux deployments in safety-critical applications, might support the addition of `pkill_on_warning`. But Lukas Bulwahn, who works on the project (but who was careful to say he doesn't speak for ELISA), [disagreed](<https://lwn.net/ml/linux-kernel/CAKXUXMx5Oi-dNVKB+8E-pdrz+ooELMZf=oT_oGXKFrNWejz=fg@mail.gmail.com/>). The right solution, he said, is to kill the system on warnings, but also to ensure that warnings are only issued in situations where things have truly gone off the rails: 

> Warnings should only be raised when something is not configured as the developers expect it or the kernel is put into a state that generally is _unexpected_ and has been exposed little to the critical thought of the developer, to testing efforts and use in other systems in the wild. Warnings should not be used for something informative, which still allows the kernel to continue running in a proper way in a generally expected environment. 

He added that being truly safe also requires ensuring that a call to `panic()` will really stop the system in all situations — something that is not as easy to demonstrate as one might think. A `panic()` call might hang trying to acquire a lock, for example. 

Christoph Leroy [said](<https://lwn.net/ml/linux-kernel/380a8fd0-d7c3-2487-7cd5-e6fc6e7693d9@csgroup.eu/>) that warnings should be handled within the kernel so that the system can keep running as well as it can. Given that, he continued, ""pkill_on_warning seems dangerous and unrelevant, probably more dangerous than doing nothing, especially as the WARN may trigger for a reason which has nothing to do with the running thread"". Popov, however, [disagreed](<https://lwn.net/ml/linux-kernel/265cd2fd-9a9b-625d-a530-299bb7433edf@linux.com/>) with the idea that one can expect all warnings to be handled properly within the kernel: 

> There is a very strong push against adding BUG*() to the kernel source code. So there are a lot of cases when WARN*() is used for severe problems because kernel developers just don't have other options. 

Indeed, his patch would, when the new option is enabled, have warnings behave in almost the same way as `BUG()` calls, which bring about the immediate end of the running process by default. As he noted, developers run into resistance when they try to add those calls because their effect is seen as being too severe. 

It's not clear that adding an option to make warnings more severe as well is the best solution to the problem. A good outcome, in the form of some movement toward better-defined notion of just what a warning means and what should happen when one is generated, could yet result from this discussion, though. Like many mechanisms in the kernel, the warning macros just sort of grew in place without any sort of overall design. Engaging in a bit of design now that there is a lot of experience with how developers actually use warnings might lead to a more robust kernel overall.
