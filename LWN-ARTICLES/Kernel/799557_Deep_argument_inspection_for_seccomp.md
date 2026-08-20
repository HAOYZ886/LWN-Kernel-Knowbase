---
title: Deep argument inspection for seccomp
url: https://lwn.net/Articles/799557/
date: "September 18, 2019"
category: "Security-seccomp"
author: "By Jake Edge September 18, 2019 LPC"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
September 18, 2019

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2019-Linux_Plumbers_Conference>)

In the [Kernel Summit track](<https://linuxplumbersconf.org/event/4/sessions/40/#all>) at the [2019 Linux Plumbers Conference](<https://linuxplumbersconf.org/event/4/>), Christian Brauner and Kees Cook led a discussion on finding a way to do deep argument inspection for seccomp filtering. Currently, seccomp filters can only look at the top-level arguments to a system call, which means that there are use cases that cannot be supported. There was a lively discussion in the session, but no definitive conclusion was reached; various ideas were considered, but none seemed to quite fit the bill. 

Cook said that the current seccomp filters can only inspect the system-call argument values; if one of those values is a pointer, dereferencing it will not work. Even if it were possible to do so, another thread could change the values after the check is done. That is a classic time-of-check-to-time-of-use (TOCTTOU) race. Programs that are using the filters would like to be able to filter based on file name arguments to restrict which files the programs can access, for example, but that is currently not possible. 

A more pressing use case is that new system calls are using an API pattern that puts various parameters (flags, in particular) into a structure, as [with `clone3()`](<https://lwn.net/Articles/792628/>), Brauner said. The address of that structure gets passed to the call along with its size, but the parameters in the structure are off-limits to filters. The idea behind the pattern is to enable additions to the API over time; the structure can be extended and the size of the structure will grow so the system call will be able to recognize when it is called with extra parameters that it does not understand. 

Both passive and active filtering of, say, `open()` calls are also affected, Cook said, so even simply logging file names as part of a passive filtering effort is not reliable. The value for the file name that the filter sees may not be the value that actually reaches the system call. The [user-space seccomp decisions feature](<https://lwn.net/Articles/756233/>) makes it possible for programs like container managers to reliably handle system calls but, since they cannot filter for only those they are interested in, they have to implement those system calls for every call; there is no way to tell the kernel to simply continue handling the system call once it has been deferred to user space. 

#### System-call flow

[ ![\[Kees Cook\]](https://static.lwn.net/images/2019/lpc-cook-sm.jpg) ](<https://lwn.net/Articles/799797/>)

The [slides [PDF]](<https://linuxplumbersconf.org/event/4/contributions/560/attachments/397/640/deep-arg-inspection.pdf>) for the session had what Cook called an "eye chart" for system-call flow that was part of the background information for attendees. After the system call enters the kernel, various [`ptrace()`](<http://man7.org/linux/man-pages/man2/ptrace.2.html>) entry hooks are called, which may block the program while another process, such as a debugger, examines, and possibly changes, the call. Anything about the call be changed, including the system-call number or its arguments; the hook can also request that the system call be skipped. 

After that, the seccomp hooks are called, which can result in a wide variety of outcomes, Cook said. They can kill the thread or process, skip the system call, log the call, send a signal to the calling process, defer the decision to user space, or generate `ptrace()` events. In the latter case, the `ptrace()` hooks may, once again, change anything, including the system-call number, which means the seccomp filter code needs to be run again. As a kind of a hack, further recursion is disallowed after one iteration of that, he said. 

Next, the actual system-call code is reached; it copies the user-space memory into kernel memory for parsing into kernel objects. At that point, the Linux security module (LSM) hooks are called, which can only make a simple accept or reject decision. If it is accepted, the system-call code then operates on the kernel objects to perform its function. Then the `ptrace()` exit hooks are called and, finally, the call returns to user space. 

Both the `ptrace()` and seccomp hooks are in the wrong place to do deterministic checks of system calls, he said. Until the arguments are copied in the system-call function itself, they can be changed by other threads, either mistakenly or as part of an attack. 

On the other hand, the LSM hooks are in the right place, but the LSM interface is not meant for system-call filtering. The LSM hooks are higher-level abstractions that are shared between system calls—the same hook can be called from multiple system calls. However, the recent addition of the [SafeSetID LSM](<https://www.kernel.org/doc/html/latest/admin-guide/LSM/SafeSetID.html>) has changed the situation somewhat; it can distinguish between system calls that seek to change user ID versus those that change the group ID. But, as yet and perhaps forever, there are no allowances for unprivileged LSMs, so the Chrome browser, for example, could not load an LSM to filter its system calls. 

Cook wondered if it makes sense to do deep inspection of system-call arguments via seccomp at all. If you really want to inspect file names or IP addresses, the LSM layer makes more logical sense since the kernel objects of interest are available there. Doing filtering based on system calls makes it possible for user space to get sloppy and only filter `open()` and ignore `rename()` for example. 

#### Other possibilities

Cook said that he explored finding a way to make an association between seccomp and an LSM of some sort. It was looking like a "really scary" solution that was overly complex with a lot of layering violations. Another way forward might be to move the seccomp hooks; the ABI says they have to be done after the `ptrace()` hooks so that means they could be pushed deeper into the system-call path. But there is a problem with that too: adding a hook to every system-call function feels completely wrong to him, for one thing. 

Another thought was to move the copying of the arguments to earlier in the system-call path; the actual system call could use that cached copy rather than doing the copying itself. The problem is that things like path-name resolution may result in different kernel objects at the two different points, which still leaves a race condition. 

Yet another idea would be to have system calls declare their argument types more completely so that the parsing of the arguments and, if needed, conversion to kernel objects could be done early in the system call path. For many system calls, this would be fairly straightforward, but path-name resolution is significantly more complex, for example. Beyond that, some system calls do things like walk lists of structures in user space, which is messy to handle; "the logic for that is really terrible". 

[ ![\[Christian Brauner\]](https://static.lwn.net/images/2019/lpc-brauner-sm.jpg) ](<https://lwn.net/Articles/799798/>)

Brauner said that they had come to the realization that allowing deep argument inspection in a generic way, for every system call, is probably not the right way forward. There is a set of system calls that are of most interest to user space for filtering for security purposes. He thought those could probably be handled separately unless someone has a great idea for how to solve the problem generically. 

He asked the assembled kernel developers if a piecemeal approach made sense by adding support for individual system calls. Aleksa Sarai said that it did make sense but he wanted to know how user space would be able to detect which system calls are supported. Cook said, "yet another problem", with a chuckle. Some way for the kernel to mark system calls as having that ability will be needed, Brauner said; that was suggested on the mailing list by Andy Lutomirski, he said. 

An attendee said that the session made it clear that the filtering is being done in the wrong place, at least for filtering based on kernel objects, such as file handles rather than path names. The LSM hooks are in the right place and have access to the right objects, so some things should be done with seccomp and others with an LSM, they said. If you want to work with file objects, for example, that would have to be done in an LSM. 

Cook agreed that filtering is happening in the wrong place, but the alternatives don't seem all that palatable either. The [Landlock LSM](<https://lwn.net/Articles/703876/>) project is working on ways to have unprivileged code be able to configure a sandbox by attaching eBPF programs to the LSM hooks. But that approach exposes LSM internals that the LSM developers are not comfortable exposing. In addition, doing system-call filtering would mean that the LSM hooks need to have a way to determine what system call invoked them, which is not something they have now (except in the limited SafeSetID case) and runs counter to the idea behind the LSM hooks. 

H. Peter Anvin said that he disagreed that seccomp filtering was being done in the wrong place. Moving the filtering deeper in the system-call path will simply expose more attack surface. He acknowledged that means that seccomp filtering cannot do all of what people would like it to do, but that isn't necessarily a problem. 

From his perspective, Brauner said that path-based filtering is not really required and that grafting it onto seccomp filtering seems wrong at some level. But there remains a need to be able to filter based on things like flags arguments that are inside structures. Others seemed to agree that handling kernel objects should be left to the LSMs, while arguments such as flags make sense for seccomp. 

An attendee wondered why LSM plus eBPF was not "just the answer". Cook said that part of the problem is that there is no unprivileged way to do either right now. It was his hope that attaching eBPF programs to LSM hooks would make its way into the kernel, thus saving seccomp from having to solve the deep argument inspection problem. There are still a lot of discussions about how that would work and both [unprivileged eBPF](<https://lwn.net/Articles/796328/>) and unprivileged LSMs have met resistance from their maintainers. 

Beyond that, attaching eBPF to LSMs gives user space a way to create "gadgets" that can be used in timing attacks of various kinds, an attendee said. That makes it hard to get the security parameters of the feature right. The LSM developers are also concerned about leaking their internal state via the eBPF programs that could be attached. 

Right now, just figuring out where the inspection would be done would be a start, Cook said. Then there are more questions of how the filtering would be hooked up to it and so on. In addition, there are the upstreaming issues, Brauner said. Toward the end, Cook said that he was hoping that someone would have a great idea that they had not thought of to solve the problem neatly, but it would appear that is not the case. It is a problem that arises frequently, though, especially in its simplest form (e.g. filtering on flag values), so it seems likely that we have not heard the last of it. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to Lisbon for LPC.]
