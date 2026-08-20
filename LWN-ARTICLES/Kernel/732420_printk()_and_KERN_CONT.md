---
title: printk() and KERN_CONT
url: https://lwn.net/Articles/732420/
date: "August 30, 2017"
category: Kernel messages; printk
author: "By Jake Edge August 30, 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
August 30, 2017

A nearly year-old "fix" to the main logging function used in the kernel, `printk()`, changed the appearance of some log messages in an unexpected way, at least for some. Messages that had appeared on a single line will now be spread over multiple lines as each call to `printk()` begins a new line in the output unless the `KERN_CONT` flag is used. That is how a comment in the kernel code says it should work, but the change was made by Linus Torvalds without any discussion or fanfare, so it took some by surprise. 

The `printk()` function is the workhorse of kernel output, for critical messages, warnings, information, and debugging. It is [used](<http://elinux.org/Debugging_by_printing>) in much the same way as [`printf()`](<http://man7.org/linux/man-pages/man3/printf.3.html>) but there are some differences. For one thing, "log levels" can be prepended to the format string to specify the severity of the message. These range from `KERN_EMERG` to `KERN_DEBUG` and can be used as follows: 

```
printk(KERN_ALERT "CPU on fire: %d\n", cpu_num);
```

The log levels are simply strings that get concatenated with the format string, thus there is no comma between them. Another difference from `printf()` is in how a format string without a newline is treated, which is what has changed. The `KERN_CONT` "log level" is meant to indicate a continuation line; a `printk()` that lacks that flag is supposed to start a new line in the log—though that hasn't always been enforced. 

Pavel Machek [posted](<https://lwn.net/Articles/732423/>) a query about that behavior on August 28\. He noted that ""printk("foo"); printk("bar"); seems to produce foo\nbar"", which was both surprising and unwelcome. That led to a bit of a [rant](<https://lwn.net/Articles/732424/>) from Torvalds, who had made the change: 

If you want to continue a line, you NEED to use KERN_CONT. 

That has always been true. It hasn't always been enforced, though. 

If you do two printk's and the second one doesn't say "I'm a continuation", the printk logic assumes you're just confused and wanted two lines. 

But, as several pointed out, that behavior only [changed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4bcc595ccd80decb4245096e3d1258989c50ed41>) relatively recently (for the 4.9 kernel released in December 2016); prior to that Machek's example would produce "foobar" as he expected. Lots of places in the kernel use `printk()` without `KERN_CONT` and expect to get output on a single line, Joe Perches [pointed out](<https://lwn.net/Articles/732433/>). Perches also complained that Torvalds had, in fact, changed longstanding behavior and was not just enforcing something that had ""always been true"". But Torvalds [pointed to](<https://lwn.net/Articles/732434/>) a [commit from 2007](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4749252776712e587c7c7619cbf0342b080c7f44>) that added the `KERN_CONT` flag, along with the following comment: 

```
/*
         * Annotation for a "continued" line of log printout (only done after a
         * line that had no enclosing \n). Only to be used by core/arch code
         * during early bootup (a continued line is not SMP-safe otherwise).
         */
```

While 2007 is not exactly "always", the comment certainly documents the intent of `KERN_CONT`, so not using it and expecting multiple calls to `printk()` to end up on the same line has not been right for nearly ten years. Torvalds was unapologetic about this recent change: 

So yes, we're enforcing it now, and we're not going back to the unenforced times, because a decade of shit has shown that people didn't do it without being forced to. 

In fact, he would like to [get rid](<https://lwn.net/Articles/732435/>) of the whole idea of continuation lines. They made some amount of sense when the output was just sent to a circular character buffer, he said, but `printk()` now has a log-based structure so continuation lines do not really work well in that environment. Beyond that, there is always the chance that some asynchronous action (e.g. an interrupt) outputs something that interferes with the single line of output. Instead, users should be marshaling their own output into single-line chunks and passing those to `printk()`, he said. 

He went on to [suggest](<https://lwn.net/Articles/732436/>) that some helper functions be added to assist in places where that marshaling is needed. Users would provide their own buffer to these routines that would then call `printk()` when they have a full line. 

That avoids the whole nasty issue with printk - printk wants to show stuff early (because _maybe_ it's critical) and printk wants to make log records with timestamps and loglevels. And printk has serious locking issues that are really nasty and fundamental. 

That set off a discussion on various ways to implement what had been suggested. Various schemes to replace `printk()` with something "smarter" were batted down quickly by Torvalds. Steven Rostedt [recommended](<https://lwn.net/Articles/732438/>) using the kernel's `seq_buf` facility that is used for tracing and implementing `/proc` files. That idea seemed to gain traction among the other thread participants (including, crucially, Torvalds). So far, no patch set along those lines has been proposed, but it seems like a promising direction. 

No matter what happens, there are going to be multiple changes to fix the output in places where `KERN_CONT` was not used but should have been. If the `seq_buf` interface is going to be used, it would make sense to do that directly, rather than add a bunch of `KERN_CONT` flags in various places. Once that is done, perhaps the existing uses of `KERN_CONT` could be tackled to get rid of as many of those as possible—leaving only those used at boot time as was originally planned. 

Clearly Torvalds doesn't think twice about breaking things internal to the kernel in order to enforce something he sees as important. Doing so silently, though, as happened here, might not have been the best approach. Had there been discussion of the patch on the mailing list, it would at least have given folks a chance to realize what was up. That might have eliminated Machek's query and perhaps reduced Torvalds's blood pressure a bit.
