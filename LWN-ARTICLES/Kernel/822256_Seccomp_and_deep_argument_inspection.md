---
title: Seccomp and deep argument inspection
url: https://lwn.net/Articles/822256/
date: "June 10, 2020"
category: "Security-seccomp"
author: "By Jake Edge June 10, 2020"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
June 10, 2020

Kees Cook has been doing some thinking about plans for new [seccomp](<https://lwn.net/Articles/656307/>) features to work on soon. There were four separate areas that he was interested in, which he [detailed](<https://lwn.net/ml/linux-kernel/202005181120.971232B7B@keescook/>) in a lengthy mid-May message on the linux-kernel mailing list. One of those features, deep argument inspection, has been [covered here before](<https://lwn.net/Articles/799557/>), but it would seem that we are getting closer to a resolution on how that all will work. 

#### Deep arguments

Seccomp filtering (or "seccomp mode 2") allows a process to filter which system calls can be made by it or its threads—it can be used to "sandbox" a program such that it cannot make calls that it shouldn't. Those filters use the "classic" BPF (cBPF) language to specify which system calls and argument values to allow or disallow. The [`seccomp()`](<https://www.man7.org/linux/man-pages/man2/seccomp.2.html>) system call is used to enable filtering mode or to load a cBPF filtering program. Those programs only have access to the _values_ of the arguments passed to the system call; if those arguments are pointers, they cannot be dereferenced by seccomp, which means that accepting or rejecting the system call cannot depend on, for example, values in structures that are passed to system calls via pointers—or even string values. 

The reason that seccomp cannot dereference the pointers is to avoid the [time-of-check-to-time-of-use](<https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use>) (TOCTTOU) race condition, where user space can change the value of what is being pointed to between the time that the kernel checks it and the time that the value gets used. But certain system calls, especially newer ones like [`clone3()`](<https://www.man7.org/linux/man-pages/man2/clone.2.html>) and [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>), have some important arguments passed in structures via pointers. These new system calls are designed with an eye toward easily adding new arguments and flags by redefining the structure that gets passed; in his email, Cook called these "extensible argument" (or EA) system calls. 

It does not make sense for seccomp to provide a mechanism to inspect the pointer arguments of every system call, he said: ""[...] the grudging consensus was reached that having seccomp do this for ALL syscalls was likely going to be extremely disruptive for very little gain"". But for the EA system calls (or perhaps only a subset of those), seccomp could copy the structure pointed to and make it available to the BPF program via its [`struct seccomp_data`](<https://elixir.bootlin.com/linux/v5.7.1/source/include/uapi/linux/seccomp.h#L51>). That would mean that seccomp would need to change to perform that copy, which would require a `copy_from_user()` call, and affected system calls would need to be seccomp-aware so that they can use the cached copy if seccomp creates one. 

There are some other wrinkles to the problem, of course. The size of the structure passed to the EA system calls may grow over time in order to add new features. If the size is larger than expected on either side (user space or kernel), finding or filling zeroes in the "extra" space is specifically designed to mean that those new features are unused (the `openat2()` man page linked above has some good information on how this is meant to work). Since user space and the kernel do not have to be in lockstep, that will allow newer user-space programs to call into an older kernel and vice versa. But that also means that seccomp needs to be prepared to handle argument sizes larger (or smaller) than "expected" and ensure that the zero-filling is done correctly. 

It gets even more complicated because different threads might have different ideas of what the EA structure size is, Cook said: 

Since there is not really any caller-based "seccomp state" associated across seccomp(2) calls, I don't think we can add a new command to tell the kernel "I'm expecting the EA struct size to be $foo bytes", since the kernel doesn't track who "I" is besides just being "current", which doesn't take into account the thread lifetime -- if a process launcher knows about one size and the child knows about another, things will get confused. 

He had suggestions of a few different possibilities to solve the problem, but seemed to prefer the zero-fill option: 

leverage the EA design and just accept anything <= PAGE_SIZE, record the "max offset" value seen during filter verification, and zero-fill the EA struct with zeros to that size when constructing the seccomp_data + EA struct that the filter will examine. Then the seccomp filter doesn't care what any of the sizes are, and userspace doesn't care what any of the sizes are. (I like this as it makes the problems to solve contained entirely by the seccomp infrastructure and does not touch user API, but I worry I'm missing some gotcha I haven't considered.) 

Others commenting also seemed to prefer that option, though Jann Horn [noted](<https://lwn.net/ml/linux-kernel/CAG48ez1LrQvR2RHD5-ZCEihL4YT1tVgoAJfGYo+M3QukumX=OQ@mail.gmail.com/>) that there is no need to zero-fill beyond the size that the kernel knows about: 

We don't need to actually zero-fill memory for this beyond what the kernel supports - AFAIK the existing APIs already say that passing a short length is equivalent to passing zeroes, so we can just replace all out-of-bounds loads with zeroing registers in the filter. The tricky case is what should happen if the userspace program passes in fields that the filter doesn't know about. The filter can see the length field passed in by userspace, and then just reject anything where the length field is bigger than the structure size the filter knows about. But maybe it'd be slightly nicer if there was an operation for "tell me whether everything starting at offset X is zeroes", so that if someone compiles with newer kernel headers where the struct is bigger, and leaves the new fields zeroed, the syscalls still go through an outdated filter properly. 

Implementing that new operation would require changes to cBPF, however, which is not going to happen, [according](<https://lwn.net/ml/linux-kernel/CAADnVQKRCCHRQrNy=V7ue38skb8nKCczScpph2WFv7U_jsS3KQ@mail.gmail.com/>) to BPF maintainer Alexei Starovoitov: ""cbpf is frozen."" An alternative would be for seccomp to switch to extended BPF (eBPF) for its filters. Using eBPF would allow the filters to perform that operation themselves without adding any new opcodes, but switching to eBPF is something that Cook hopes to avoid. As he explained in a [message](<https://lore.kernel.org/netdev/CAGXu5jKv3QFVKLhok1JWiPamE0b4CqLTO-hx8sP0KWED921=6w@mail.gmail.com/>) back in 2018, eBPF is something of fast-moving target, which worries him from a security standpoint: ""[...] I want absolutely zero surprises when it comes to seccomp"". Beyond that, eBPF would add a lot more code for the seccomp filter to interact with in potentially dangerous ways. 

Aleksa Sarai, who is the developer behind the EA scheme, generally [agreed](<https://lwn.net/ml/linux-kernel/20200519070929.55r3xvybjg6nnbsj@yavin.dot.cyphar.com/>) with Cook's plan for handling those structures, but he raised another point. The structures may contain pointers—those cannot be dereferenced by seccomp either, of course. Should something be done so that the filters can access that data as well? When these "nested pointers" came up in another discussion, Linus Torvalds made it abundantly clear that he thinks that is not a problem that the kernel should deal with at all. 

#### Less-deep arguments

A few days after his original post, Cook [posted](<https://lwn.net/ml/ksummit-discuss/202005200917.71E6A5B20@keescook/>) an item on the ksummit-discuss mailing list to suggest that there be a session at the (virtual) Kernel Summit in August to discuss these seccomp issues. Torvalds [acknowledged](<https://lwn.net/ml/ksummit-discuss/CAHk-=wierGOJZhzrj1+R18id-WdfmK=eWT9YfWdCfMvEO+jLLg@mail.gmail.com/>) that this kind of system call exists, but did not think there was much to discuss with regard to seccomp: 

So I am not in the least interested in some kind of general discussion about system calls with "pointers to pointers". They exist. Deal with it. It's not in the least an interesting issue, and no, we shouldn't make seccomp and friends incredibly more complicated for it. 

[...] And if you have some actual and imminent real security issue, you mention _that_ and explain _that_, and accept that maybe you need to do that expensive emulation (because the kernel people just don't care about your private hang-ups) or you need to explain why it's a real issue and why the kernel should help with your odd special case. 

Cook seemed somewhat relieved in his [response](<https://lwn.net/ml/ksummit-discuss/202005201151.AFA3C9E@keescook/>): 

Perhaps the question is "how deeply does seccomp need to inspect?" and maybe it does not get to see anything beyond just the "top level" struct (i.e. struct clone_args) and all pointers within THAT become opaque? That certainly simplifies the design. 

Christian Brauner, who has also been doing a lot of development in these areas, [agreed](<https://lwn.net/ml/ksummit-discuss/20200520202401.s22hstao4kzr5uma@wittgenstein/>) that the filters could likely live without the ability to chase pointers any further than the top level. Sarai would [like to see](<https://lwn.net/ml/ksummit-discuss/20200522040606.ec64dvpbldn3ufh3@yavin.dot.cyphar.com/>) there at least be a path forward if requirements of that sort do arise, but seemed willing to keep things simple for now—perhaps forever. 

#### io_uring

In his message on linux-kernel, Horn raised an interesting point for seccomp developers: handling [io_uring](<https://lwn.net/Articles/776703/>). Since its introduction in early 2019, io_uring has [rapidly added features](<https://lwn.net/Articles/810414/>) that effectively allow routing around the normal system-call entry path, while still performing the actions that a seccomp filter might be trying to prevent. 

io_uring is growing support for more and more syscalls, including things like openat2, connect, sendmsg, splice and so on, and that list is probably just going to grow in the future. If people start wanting to use io_uring in software with seccomp filters, it might be necessary to come up with some mechanism to prevent io_uring from permitting access to almost everything else... 

Obviously, the filters could simply disallow the io_uring system calls entirely, but that may be problematic down the road. Sarai [agreed](<https://lwn.net/ml/linux-kernel/20200519024846.b6dr5cjojnuetuyb@yavin.dot.cyphar.com/>) that it is something that may need some attention. Cook [said](<https://lwn.net/ml/linux-kernel/202005181630.60E58CA0C5@keescook/>) that he needed to look more closely at io_uring: ""I thought this was strictly for I/O ... like it's named"". Trying to filter based on the arguments to the io_uring system calls will be a difficult problem to solve, since the actual commands and their arguments are buried inside a ring buffer that lives in an [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) region shared between the kernel and user space. Chasing pointers in that environment seems likely to require eBPF—or even stronger medicine. 

It would seem that a reasonable path for inspecting the first level of structure "arguments" to _some_ system calls has been identified. `clone3()` and `openat2()` are obvious candidates, since their flag arguments, which will help seccomp filters determine if the call is "reasonable" under the rules of the sandbox, live in such structures. On the other hand, complex, multiplexing system calls like [`ioctl()`](<https://www.man7.org/linux/man-pages/man2/ioctl.2.html>) and [`bpf()`](<https://www.man7.org/linux/man-pages/man2/bpf.2.html>) were specifically mentioned as system calls that would not make sense to try to add the pointer-chasing feature. Though Cook did not put any timetable on his plans, one might think we will see this feature sometime before the end of the year.
