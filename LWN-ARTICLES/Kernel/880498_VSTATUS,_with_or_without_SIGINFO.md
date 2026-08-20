---
title: "VSTATUS, with or without SIGINFO"
url: https://lwn.net/Articles/880498/
date: "January 6, 2022"
category: Signal handling
author: "By Jonathan Corbet January 6, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 6, 2022

The Unix [signal interface](<https://man7.org/linux/man-pages/man7/signal.7.html>) is complex and hard to work with; some developers have [argued](<https://lwn.net/Articles/414618/>) that its design is "unfixable". So when Walt Drummond [proposed](<https://lwn.net/ml/linux-kernel/20220103181956.983342-1-walt@drummond.us/>) increasing the number of signals that Linux systems could manage, eyebrows could be observed at increased altitude across the Internet. The proposed increase seems unlikely to happen, but the underlying goal — to support a decades-old feature from other operating systems — may yet become a reality. 

The kernel is able to support up to 64 different signal types, which seems like a fair number, but all 64 are taken, on some architectures at least. That makes it impossible to add new signal types to Linux. Drummond sought to address that problem by raising the limit to 1024, which would surely be enough for all time. Raising the limit requires making some subtle changes to the user-space API (putting a larger signal mask into the information passed to realtime signal handlers, for example) that have the possibility of breaking applications, which means that extra scrutiny would be required. But that, it seems, is what would be needed to be able to add more signals. 

Developers immediately wanted to know why there was a need to add signals, and urged Drummond to find an alternative if possible. As Eric Biederman [put it](<https://lwn.net/ml/linux-kernel/87iluzidod.fsf@email.froward.int.ebiederm.org/>): 

> Please let's not expand the number of signals supported if there is any alternative. Signals only really make sense for supporting existing interfaces. For new applications there is almost always something better. 

So why is there a need to add new signals? The answer has roots in ancient history, well before the creation of Linux. The [TOPS-10](<https://en.wikipedia.org/wiki/TOPS-10>) operating system was developed by Digital Equipment Corporation in the 1970s; one feature it offered was to print a line of status information (what was running, CPU time consumed, etc.) when the user typed control-T. It was a quick way for users to verify that the system was still alive and making progress on whatever task they were running. This feature found its way into VMS later on, and it still exists today in a number of BSD variants. Linux, however, does not have this feature — in that form, anyway. 

Within the kernel, there are two aspects to supporting this feature, which goes by the name VSTATUS. The first is to recognize the status request in the terminal driver; this is done using the same logic that recognizes other control characters, such as control-C to send a `SIGINT` signal to the running process. The kernel would then respond to the keystroke in two ways: 

  * The kernel will print a status line directly to the terminal with generic information about the running process and the state of the system. 
  * A `SIGINFO` signal is sent to the process running on the terminal at the time. Applications can catch this signal to print some status information of their own; a copy application could tell the user how far the copy has progressed, for example. 

The Linux kernel doesn't implement VSTATUS, so it won't do either of the above things. Adding the ability to print a status line to the terminal driver is not that hard; neither is sending a signal in response to an event. The problem is that Linux does not implement `SIGINFO`, so that signal would need to be added and, as noted above, there is no room to add new signals. Thus Drummond's patch set. 

This is not the first time that VSTATUS support has been requested; the issue also [came up in 2014](<https://lore.kernel.org/lkml/1415200663.3247743.187387481.75CE9317@webmail.messagingengine.com/>) and [again in 2019](<https://lwn.net/ml/linux-kernel/20190625161153.29811-1-ar@cs.msu.ru/>) — and undoubtedly other times as well. In 2019, Arseny Maslennikov tried to get around the signal limitation by defining `SIGINFO` as a synonym for `SIGPWR`, which was meant to (depending on who is reminiscing at the time) indicate that either [a power failure was impending](<https://lwn.net/ml/linux-kernel/20190625213215.GB3116@mit.edu/>) or [power had been restored](<https://lore.kernel.org/lkml/20141110142200.676b5d04@alan.etchedpixels.co.uk/>). Either way, the signal is delivered only to the `init` process, and it tends to be little used on current systems. Repurposing it for VSTATUS requires changing the default action for `SIGPWR` to "ignore" but otherwise shouldn't prove disruptive to user space. 

Or so the developers would hope. Real-world uses of `SIGPWR` are rare, but they do exist; as Ted Ts'o [pointed out](<https://lwn.net/ml/linux-kernel/YdSzjPbVDVGKT4km@mit.edu/>), systemd can be made to respond to it, for example. Changing the default handling of `SIGPWR` would _probably_ not break any user-space applications, but it is hard to know that for sure. If something does break, it could show up years later when the change makes it into an enterprise kernel; at that point, the problem would be nearly impossible to fix to everybody's satisfaction. So it is unsurprising that kernel developers are reluctant to make a change like this. 

Over the years, there have also been questions about whether the feature is really needed or not. As Greg Kroah-Hartman [pointed out](<https://lwn.net/ml/linux-kernel/20190730161940.GA15798@kroah.com/>) in 2019, the kernel's "[magic SysRq](<https://www.kernel.org/doc/html/latest/admin-guide/sysrq.html>)" feature does everything VSTATUS does (and a lot more). But magic SysRq only works on the system console; as Biederman noted in the above-linked message, a ""persuasive case"" could be made for the utility of this feature for users interacting with systems over SSH, for example. So a use case for VSTATUS does appear to exist. 

Given that, what is to be done? Ts'o's message above included a suggestion: implement VSTATUS as far as printing the status line from the kernel, but don't bother sending the `SIGINFO` signal to user space. That eliminates the need to add a new signal (or repurpose an old one), which is where all the problems arise. This implementation would deprive user space of the ability to add its own status information, but the number of programs that have ever implemented `SIGINFO` handling is quite small; Drummond [said](<https://lwn.net/ml/linux-kernel/CADCN6nzT-Dw-AabtwWrfVRDd5HzMS3EOy8WkeomicJF07nQyoA@mail.gmail.com/>) that only `sleep`, `dd`, and `ping` have such support, ""so it's not like there's a vast hole in the tooling or something, nor is there a large legacy software base just waiting for SIGINFO to appear"". He readily [agreed](<https://lwn.net/ml/linux-kernel/CADCN6nz0ih2k7-LB9D3qJjQ9Dv5QAkn7KC9Ci-qcbMHTG7_F+A@mail.gmail.com/>) to leave out the `SIGINFO` part. 

The followup patch set without `SIGINFO` has not been posted as of this writing. Once it arrives, there should not be a great deal of opposition to its merging into the mainline; at that point, Linux will have most of the VSTATUS functionality that is offered by the BSDs. If it turns out later on that somebody really needs the `SIGINFO` part, that whole problem can be reconsidered. Meanwhile, though, the kernel community is happy to kick that can further down the road.
