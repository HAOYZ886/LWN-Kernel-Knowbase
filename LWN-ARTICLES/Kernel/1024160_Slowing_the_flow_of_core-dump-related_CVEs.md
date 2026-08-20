---
title: "Slowing the flow of core-dump-related CVEs"
url: https://lwn.net/Articles/1024160/
date: "June 6, 2025"
category: "Releases-6.16; Security-Vulnerabilities"
author: "By Jonathan Corbet June 6, 2025"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 6, 2025

The 6.16 kernel will include a number of changes to how the kernel handles the processing of core dumps for crashed processes. Christian Brauner [explained](<https://mastodon.social/@brauner/114592290899392625>) his reasons for doing this work as: ""Because I'm a clown and also I had it with all the CVEs because we provide a **** API for userspace"". The handling of core dumps has indeed been a constant source of vulnerabilities; with luck, the 6.16 work will result in rather fewer of them in the future. 

#### The problem with core dumps

A core dump is an image of a process's data areas — everything except the executable text; it can be used to investigate the cause of a crash by examining a process's state at the time things went wrong. Once upon a time, Unix systems would routinely place a core dump into a file called `core` in the current working directory when a program crashed. The main effects of this practice were to inspire system administrators worldwide to remove `core` files daily via `cron` jobs, and to make it hazardous to use the name `core` for anything you wanted to keep. Linux systems can still create `core` files, but are usually configured not to. 

An alternative that _is_ used on some systems is to have the kernel launch a process to read the core dump from a crashing process and, presumably, do something useful with it. This behavior is configured by writing an appropriate string to [the `core_pattern` sysctl knob](<https://docs.kernel.org/admin-guide/sysctl/kernel.html#core-pattern>). A number of distributors use this mechanism to set up core-dump handlers that phone home to report crashes so that the guilty programs can, hopefully, be fixed. 

This is the ""**** API"" referred to by Brauner; it indeed has a number of problems. For example, the core-dump handler is launched by the kernel as a user-mode helper, meaning that it runs fully privileged in the root namespace. That, needless to say, makes it an attractive target for attackers. There are also a number of race conditions that emerge from this design that have led to vulnerabilities of their own. 

See, for example, [this recent Qualys advisory](<https://lwn.net/ml/all/20250529171556.GA9260@localhost.localdomain>) describing a vulnerability in Ubuntu's [`apport`](<https://wiki.ubuntu.com/Apport>) tool and the [`systemd-coredump`](<https://www.freedesktop.org/software/systemd/man/latest/systemd-coredump.html>) utility, both of which are designed to process core dumps. In short, an attacker starts by running a setuid binary, then forcing it to crash at an opportune moment. While the core-dump handler is being launched (a step that the attacker can delay in various ways), the crashed process is killed outright with a `SIGKILL` signal, then quickly replaced by another process with the same process ID. The core-dump handler will then begin to examine the core dump from the crashed process, but with the information from the replacement process. 

That process is running in its own attacker-crafted namespace, with some strategic environmental changes. In this environment, the core-dump handler's attempt to pass the core-dump socket to a helper can be intercepted; that allows said process to gain access to the file descriptor from which the core dump can be read. That, in turn, gives the attacker the ability to read the (original, privileged) process's memory, happily pillaging any secrets found there. The example given by Qualys obtains the contents of `/etc/shadow`, which is normally unreadable, but it seems that SSH servers (and the keys in their memory) are vulnerable to the same sort of attack. 

Interested readers should consult the advisory for a much more detailed (and coherent) description of how this attack works, as well as information on some previous vulnerabilities in this area. The key takeaways, though, are that core-dump handlers on a number of widely used distributions are vulnerable to this attack, and that reusable integer IDs as a way to identify processes are just as much of a problem as the pidfd developers have been saying over the years. 

#### Toward a better API

The solution to this kind of race condition is to give the core-dump handler a way to know that the process it is investigating is, indeed, the one that crashed. The 6.16 kernel contains two separate changes toward that goal. The first is [this patch from Brauner](<https://git.kernel.org/linus/b5325b2a270f>) adding a new format specifier ("`%F`") for the string written to `core_pattern`. This specifier will cause the core-dump handler to be launched with a pidfd identifying the crashed process installed as file descriptor number three. Since it is a pidfd, it will always refer to the intended process and cannot be fooled by process-ID reuse. 

This change makes it relatively easy to adapt core-dump handlers to avoid the most recently identified vulnerabilities; it has already been backported to [a recent set of stable kernels](<https://lwn.net/Articles/1023794/>). But it does not change the basic nature of the `core_pattern` API, which still requires the launch of a new, fully privileged process to handle each crash. It is, instead, a workaround for one of the worst problems with that API. 

The longer-term fix is [this series from Brauner](<https://lwn.net/ml/all/20250507-work-coredump-socket-v4-0-af0ef317b2d0@kernel.org>), which was also merged for 6.16. It adds a new syntax to `core_pattern` instructing the kernel to write core dumps to an existing socket; a user-space handler can bind to that socket and accept a new connection for each core dump that the kernel sends its way. The handler must be privileged to bind to the socket, but it remains an ordinary process rather than a kernel-created user-mode helper, and the process that actually reads core dumps requires no special privileges at all. So the core-dump handler can bind to the socket, then drop its privileges and sandbox itself, closing off a number of attack vectors. 

Once a new connection has been made, the handler can obtain a pidfd for the crashed process using the `SO_PEERPIDFD` request for [`getsockopt()`](<https://man7.org/linux/man-pages/man2/setsockopt.2.html>). Once again, the pidfd will refer to the actual crashed process, rather than something an attacker might want the handler to treat like the crashed process. The handler can pass the new `PIDFD_INFO_COREDUMP` option to the [`PIDFD_GET_INFO` `ioctl()` command](<https://lwn.net/Articles/992991/>) to learn more about the crashed process, including whether the process is, indeed, having its core dumped. There are, in other words, a couple of layers of defense against the sort of substitution attack demonstrated by Qualys. 

The end result is a system for handling core dumps that is more efficient (since there is no need to launch new helper processes each time) and which should be far more resistant to many types of attacks. It may take some time to roll out to deployed systems, since this change seems unlikely to be backported to the stable kernels (though distributors may well choose to backport it to their own kernels). But, eventually, this particular source of CVEs should become rather less productive than it traditionally has been.
