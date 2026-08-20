---
title: "Back to the drawing board for utrace?"
url: https://lwn.net/Articles/371210/
date: "January 27, 2010"
category: Utrace
author: "By Jake Edge January 27, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
January 27, 2010

The utrace tracing framework has had a tortuous path towards the mainline, but it always seemed like it was headed that direction. Over the past week or so, things have gotten rather murkier for the mainline inclusion of utrace. Linus Torvalds made a [pronouncement](<http://lwn.net/Articles/370715/>) that would seem to leave SystemTap without a future in the mainline--something that many had suspected for a while--but also put the future of utrace in doubt. Further discussion may have provided a way forward, but, at least in its current form, mainline utrace seems very unlikely. 

The discussion resulted from a [request](<https://lwn.net/Articles/371347/>) by Frank Ch. Eigler to include utrace into linux-next. That led to a discussion about whether it was ready for linux-next--because it was likely to be merged in the next release cycle--or whether it should spend some time in another tree. Since an earlier version of utrace was in Andrew Morton's -mm tree, that was a potential path. Morton [said](<https://lwn.net/Articles/371356/>) that utrace ""didn't break anything"", but: 

I still don't think I've seen a really compelling reason for merging it. At least, I wouldn't be able to explain why we did it. But presumably there _are_ such reasons, because it was a lot of development work. 

Someone please sell this to us. 

Morton also [dredged up](<https://lwn.net/Articles/371357/>) a response he had gotten from Oleg Nesterov the last time he asked, which listed various potential uses for utrace. In-kernel uses for utrace are important--new features are rarely merged without one--and an earlier [utrace merge attempt](<http://lwn.net/Articles/325180/>) ran into opposition because it lacked one. [This time around](<http://lwn.net/Articles/364584/>), Nesterov and Roland McGrath included a rewrite of the `ptrace()` system call using utrace as part of the patch submission. It was hoped that rewriting the notoriously ugly `ptrace()` code using the cleaner utrace API would be the last hurdle for inclusion into the mainline. 

But, replacing the guts of the `ptrace()` call, even though it may clean things up, is controversial. `ptrace()` is part of the kernel ABI that must be maintained--ugly or not--but cleaning it up is not without its risks, as Morton [points out](<https://lwn.net/Articles/371368/>): 

ptrace is a nasty, complex part of the kernel which has a long history of problems, but it's all been pretty quiet in there for the the past few years. This leads one to expect that a rip-out-n-rewrite is a high-risk prospect. So, quite reasonably, one looks for a good reason for taking such risk. 

The risk is small, though, [according to Eigler](<https://lwn.net/Articles/371373/>), because ""this code has been deployed in fedora and rhel for several *years*, with millions of users. It's not some rickety experiment."" Eigler also added to Nesterov's list of utrace uses as SystemTap's user-space probing is based on utrace. But SystemTap and one of the other potential uses on that list, namely reworking seccomp to use utrace, are what set Torvalds off: 

So if things like system tap and "security models that go behind the kernel by tying into utrace" are the reasons for utrace, color me utterly uninterested. In fact, color me actively hostile. I think that's the worst possible situation that we'd ever be in as kernel people (namely exactly the "do things in kernel space by hiding behind utrace without having kernel people involved") 

Torvalds's complaint stems from the fact that utrace provides no user-space interface at all. It is purely an internal kernel API that is meant to be used by kernel code like the `ptrace()` rewrite, but also for kernel modules, which is part of what worries Torvalds. It provides lots of hooks that can be used by ""random crazy out-of-tree crap"", but doesn't provide any benefit to user space at all, he [said](<https://lwn.net/Articles/371380/>): 

If somebody were to argue that "this is a simple series of patches to clean up ptrace and make it possible to strace a debugged process", then that would have been different. That's not what you or others have been doing. You've been pushing exactly the _reverse_ of that, namely how great it is for some random totally new features that I'm convinced aren't even used by a lot of people. 

One of the biggest problems with `ptrace()` is its signal-oriented interface. Programs using `ptrace()` act as the parent process of the tracee and must use `wait()` to detect state changes. For that reason, there can only be one `ptrace()` active for a particular process. So an `strace` of a program that is being debugged with `gdb` will not succeed. The `ptrace()` implementation using utrace would change that, but not directly, as there would still need to be a kernel piece that attached another utrace engine. 

An [in-kernel `gdb` "stub"](<http://lwn.net/Articles/364268/>) using utrace--floated as an RFC back in November--could provide that kernel piece, but was met with a fair amount of resistance when it was proposed. The limitation that `ptrace()` imposes is seen as something that could, perhaps should, be lifted, but adding a relatively large, kernel-only API to do that is excessive. As Torvalds [puts it](<https://lwn.net/Articles/371390/>): 

Maybe somebody would be interested in trying to take the utrace improvements, and scaling down what they promise, and ignoring all input except for "I want to strace and gdb at the same time". 

So stop the crazy "new kernel interfaces" crap. Stop the crazy "maybe we can use it for ftrace and generic user event tracing too". Stop the crazy. 

The elephant in the room, of course, is SystemTap. It creates, builds, and loads kernel modules for doing its tracing, and uses utrace for the user-space tracing. That model is not popular with most kernel developers, especially for an out-of-tree solution--the APIs that it relies on are far too volatile. SystemTap must be updated when those interfaces change, and all of the previous versions must be maintained so that SystemTap can still be used with older kernels. Because of that, SystemTap may be out-of-sync with development kernels, which makes its utility for kernel hackers quite small. 

The utrace proponents are pushing it as something useful in its own right, completely separate from its use in SystemTap, but one gets the sense that many of the kernel developers aren't quite buying that. Ted Ts'o tries to [explain](<https://lwn.net/Articles/371394/>) his concerns to Eigler 

[...] utrace doesn't export a syscall (which is an ABI that we are willing to promise will be stable), but rather a set of kernel API's (which we never promise to be stable), and the fact that there will be out-of-tree programs that are going to be trying to depend on that interface (much like Systemtap does today when it creates kernel modules) [...] 

He goes on to compare the situation to that of the NVIDIA graphics drivers, which leads Kyle Moffett to propose a [variation on Godwin's law](<https://lwn.net/Articles/371395/>): ""As an LKML discussion grows longer, the probability of an unfavorable comparison involving nVidia or Microsoft approaches 1."" More to the point, though, Moffett said he was uninterested in SystemTap: 

I'm interested in things like the ability to stack gdb with strace, the RFC gdb-stub posted a week ago, etc. None of those abilities would be out-of-tree modules at all [...] 

Ts'o [sees](<https://lwn.net/Articles/371396/>) those features as potentially useful, but points out that they should be submitted with utrace for review. It may be that utrace in its present form does not survive that review: 

So what should be reviewed is utrace *plus* these other userland interfaces, which may get critiqued and improved, and utrace patches can be reviewed in light of these new features. But be warned.... if it turns out that only 30% of utrace is only needed to support gdb stacking with strace, etc., the other 70% will likely get ejected and the utrace patches streamlined to support these in-tree users. 

Without an in-tree "killer feature" that only utrace can provide, there is going to be resistance to merging such an easily-abused API. Several suggestions were made--notably by Torvalds and Ingo Molnar--to enhance `ptrace()` itself to support some new features (such as multiple active calls or the ability to read/write more than a word at a time between the two processes), but that would mean scrapping much or all of the utrace work. Nesterov and McGrath, who are the `ptrace()` maintainers, have been largely silent throughout the discussion, but, previously, they have made it clear that they would much rather work with the utrace-based `ptrace()` implementation. So it is unclear when or if enhancements to the current code might happen. 

Without utrace, SystemTap will have to find other ways to hook user space, but that doesn't really faze the kernel developers--particularly after Torvalds's unequivocal rejection of that approach--as there are other tracing solutions in the pipeline. Ftrace and perf events are slowly building capabilities, and are doing so in-tree. They are likely to grow the needed features to support kernel and user-space tracing _a la_ SystemTap (and DTrace). Molnar specifically [invites](<https://lwn.net/Articles/371397/>) the SystemTap developers to collaborate: 

Also, if any systemtap person is interested in helping us create a more generic filter engine out of the current ftrace filter engine (which is really a precursor of a safe, sandboxed in-kernel script engine), that would be excellent as well. Right now we support simple C-syntax expressions like: 

```
perf record -R -f -e irq:irq_handler_entry --filter 'irq==18 || irq==19'
```

More could be done - a simple C-like set of function perhaps - some minimal per probe local variable state, etc. (perhaps even looping as well, with a limit on number of [predicate] executions per filter invocation.) 

It is unfortunate, in many ways, that SystemTap has gotten to this point. While it is possible that Torvalds could change his mind, he and other kernel developers find the new tracing features to be ""a million times superior"" to SystemTap. That could leave Red Hat holding the SystemTap bag for quite some time to come, as it will need to support it for existing, and likely future, RHEL versions. It is interesting to note that this alternate solution, based on Ftrace, etc., is also largely coming out of Red Hat. 

It seems possible that utrace will be a casualty here as well. By incorporating features that were needed for SystemTap, and not providing a user-space interface, it tried to both do too much and too little. There are some potential ways forward, but its unclear whether they will be pursued. Torvalds [points to](<http://lwn.net/Articles/370998/>) the realtime tree as an example of how to get "crazy" things merged: 

Yeah, it's taken them years, and they still have out-of-tree stuff. And yeah, they had to change some things to make them more palatable to the mainline kernel - the whole fundamental raw spinlock change is just the most recent example of that. 

But on the whole, I think it's actually worked out pretty well for them. I think the mainline kernel has improved in the process, but I also suspect that _their_ RT patches have also improved thanks to having to make the work more palatable to people like me who don't care all that deeply about their particular flavor of crazy. 

There are definitely lessons here, but the standard ones don't seem to apply. SystemTap and utrace were developed in the open, as free software from the outset, and were fairly often discussed on linux-kernel. SystemTap in particular was regularly criticized, to seemingly no avail. The biggest lesson--and the hardest to learn, especially after a feature has shipped--may be that ignoring the advice and complaints of the kernel developers is likely to come back and bite in the end. It is not terribly surprising, really, but that seems to be what is happening here.
