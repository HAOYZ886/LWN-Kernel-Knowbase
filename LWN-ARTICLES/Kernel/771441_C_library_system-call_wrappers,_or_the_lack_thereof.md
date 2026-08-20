---
title: "C library system-call wrappers, or the lack thereof"
url: https://lwn.net/Articles/771441/
date: "November 12, 2018"
category: "Development model-User-space ABI"
author: "By Jonathan Corbet November 12, 2018"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 12, 2018

User-space developers may be accustomed to thinking of system calls as direct calls into the kernel. Indeed, the first edition of _The C Programming Language_ described `read()` and `write()` as ""a direct entry into the operating system"". In truth, user-level "system calls" are just functions in the C library like any other. But what happens when the developers of the C library refuse to provide access to system calls they don't like? The result is an ongoing conflict that has recently flared up again; it shows some of the difficulties that can arise when the system as a whole has no ultimate designer and the developers are not talking to each other. 

Calling into the kernel is not like calling a normal function; a special trap into the kernel must be triggered with the system-call arguments placed as the kernel expects. At a minimum, the system-call "wrapper" provided by the C library must set up this trap. In many cases, more work than that is required; the functionality provided by the kernel does not always exactly match what the application (or the relevant standards) will expect. Features like POSIX threads further complicate the situation. The end result is that a lot of work can be happening between the application and the kernel when a system call is made. Doing that work is, in most cases, delegated to the C library. 

#### System calls in glibc

Many Linux systems use the GNU C Library (glibc) in this role; glibc is often thought of as _the_ Linux C library. When the kernel developers add a new system call, it is thus natural to expect that a corresponding wrapper will show up in glibc, but there is no guarantee that this will ever happen. The addition of wrappers to glibc is often slow and, in some cases, the glibc developers have refused to add the wrappers at all. In such cases, user-space developers must fall back on [`syscall()`](<http://man7.org/linux/man-pages/man2/syscall.2.html>) to access that functionality, an approach that is both non-portable and error-prone. 

Recently, frustration with this situation led Daniel Colascione to [ask](<https://lwn.net/ml/linux-kernel/CAKOZuesB4R=dCz4merWQN0FSCGrXmOgUUr4ienSbStBJguNv8g@mail.gmail.com/>): 

Now that glibc is basically not adding any new system call wrappers, how about publishing an "official" system call glue library as part of the kernel distribution, along with the uapi headers? I don't think it's reasonable to expect people to keep using syscall(__NR_XXX) for all new functionality, especially as the system grows increasingly sophisticated capabilities (like the new mount API, and hopefully the new process API) outside the strictures of the POSIX process. 

It is worth noting, as [Michael Kerrisk did](<https://lwn.net/ml/linux-kernel/bbc12da5-830e-99a7-95e3-d9da42947dc9@gmail.com/>), that it's not really true that glibc is no longer adding wrappers; quite a few have found their way into recent releases. But there are some notable exceptions, the most glaring of which is probably `gettid()`, which has been [under discussion](<https://sourceware.org/bugzilla/show_bug.cgi?id=6399>) for over a decade with no real resolution in sight. Kerrisk suggested that, in most cases, the problem was simply a lack of developers on the glibc side and [said](<https://lwn.net/ml/linux-kernel/3664a508-ca74-4ff0-39a6-34543194a24e@gmail.com/>) that kernel developers should take more responsibility for the creation of glibc wrappers for new system calls: 

A converse question that one could ask is: why did a culture evolve whereby kernel developers don't take responsibility for working with the major libc to ensure that wrappers are added as part of the job of adding each new system call? 

Glibc developer Florian Weimer [stated clearly](<https://lwn.net/ml/linux-kernel/87tvknvmud.fsf@oldenburg.str.redhat.com/>) that it's not just a matter of developer time, though: ""It's not a matter of resources or lack thereof"". In another message he [explained](<https://lwn.net/ml/linux-kernel/877ehjx447.fsf@oldenburg.str.redhat.com/>) why many system calls lack glibc wrappers, with a number of specific examples. ""A lot of the new system calls lack clear specifications or are just somewhat misdesigned"". In other cases, new system calls — such as `renameat2()` — use names that glibc had already used for other functions. Reasons vary, but the end result is the same for a number of system calls: no glibc wrappers to go along with them. 

According to Colascione (and others like [Willy Tarreau](<https://lwn.net/ml/linux-kernel/20181111111143.GB4189@1wt.eu/>)), the proper answer is to [provide low-level system-call wrappers](<https://lwn.net/ml/linux-kernel/CAKOZues5SEESpJU=6MDTrPXTA1KTZFGNQE4Lw4t0fO-WBTU62w@mail.gmail.com/>) with the kernel itself: 

These objections illustrate my point. glibc development is not the proper forum for raising post-hoc objections to system call design. Withholding wrappers will not un-ship these system calls. Applications are already using them, via syscall(2). Developers and users would be better served by providing access to the system as it is, with appropriate documentation caveats, than by holding out for some alternate and more ideal set of system calls that may or may not appear in the future. 

In this view, glibc would retain all of the higher-level C-library functions, ceding only the system-call wrappers to this new library. But, [according to Weimer](<https://lwn.net/ml/linux-kernel/87zhufvntw.fsf@oldenburg.str.redhat.com/>), it's not so simple: circumventing glibc for system calls would break features, many associated with threading. Or, as Zack Weinberg [put it](<https://lwn.net/ml/linux-kernel/CAKCAbMiHC9r54h=XeW7CkBZ1Z5eHr9MPH3Rn7KTc9DjoHG=8UA@mail.gmail.com/>): ""The trouble is that 'raw system call wrappers and arcane kernel-userland glue' turns out to be a lot more code, with a lot more tentacles in both directions, than you might think"". It's not just a matter of breaking things out into a separate library. 

#### Overcoming the impasse

Arguably, what this whole discussion is really showing is that there need to be better lines of communication between kernel and C-library developers. It takes developers from both groups to actually make a feature available to user space after all; it would make sense for kernel developers — who are not always known for the best API designs — to talk more with the library developers who must actually support an API for application developers. Those communications are better now than they were some years ago, but one could argue that this is a low bar that has not been surmounted by much. 

One complication there is that glibc is not the only C library that runs over the Linux kernel; it's not even the most popular one if one looks at the number of installed copies — that title surely belongs to the [bionic](<https://android.googlesource.com/platform/bionic>) library used by Android. The Linux community would be well served by a forum where developers from all C libraries could interact with kernel developers to address API problems before they are set into stone. The linux-api mailing list ostensibly serves that purpose now, but it is underused even before considering the absence of C-library developers there. 

Once upon a time, all operating systems had an overall architect who would be responsible for ensuring coordination between the various layers of the system, but Linux lacks such a person. So developers have to find ways to coordinate on their own. Arguably, one place where this should be happening is the [Linux Plumbers Conference](<https://linuxplumbersconf.org>), which starts November 13 in Vancouver. There is indeed [a relevant session](<https://linuxplumbersconf.org/event/2/contributions/149/>) on the agenda, but it's not clear how many of the necessary developers will be there. 

Free-software projects tend to value their independence; their developers have little time for others who would tell them what to do. But few projects truly stand alone. Whenever developers decide to cooperate more fully with related projects, the result tends to be better software for the community as a whole. The design and delivery of system calls would appear to be one of those places where a higher level of communication and cooperation would be a healthy thing. That, rather than trying to absorb low-level wrappers into the kernel project, seems like the proper long-term solution to this problem.
