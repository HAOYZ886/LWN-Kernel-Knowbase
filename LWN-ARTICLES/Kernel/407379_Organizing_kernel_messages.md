---
title: Organizing kernel messages
url: https://lwn.net/Articles/407379/
date: "September 29, 2010"
category: Messages
author: "By Jonathan Corbet September 29, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 29, 2010

In a previous life, your editor developed Fortran code on a VAX/VMS system. Every message emitted by VMS came decorated with a unique identifier which could be used to look it up in a massive blue binder, yielding a few paragraphs of (hopefully) helpful text on what the message actually meant. Linux has no analogous mechanism, but that is not the result of a lack of attempts. A talk at LinuxCon Japan detailed a new approach to organized kernel messaging which, its authors hope, has a better chance of making it into the mainline. 

Andrew Morton recently [described](<http://lwn.net/Articles/392177/>) the kernel's approach to messaging this way: 

The kernel's whole approach to messaging is pretty haphazard and lame and sad. There have been various proposals to improve the usefulness and to rationally categorise things in way which are more useful to operators, but nothing seems to ever get over the line 

At LinuxCon Japan, Hisashi Hashimoto described an effort which, he hopes, will get over the line. To that end, he and others have examined previous attempts to bring order to kernel messaging. Undeterred, they have pushed forward with a new project; he then introduced Kazuo Ito who discussed the actual work. 

Attempts to regularize kernel messaging usually involve either attaching an identifier to kernel messages or standardizing the message format in some way. One thing that Ito-san noted at the outset is that any scheme requiring wholesale changes to `printk()` lines is probably not going to get very far. There are over 75,000 such lines in the kernel, many of them buried within macros; there is no practical way to change them all. Other wrapper functions, such as `dev_printk()`, complicate the situation further. So any change will have to be effected in a way which works with the existing mass of `printk()` calls. 

A few approaches were considered. One would be to create a set of wrapper macros which would format message identifiers and pass them to `printk()`; the disadvantage of this method, of course, is that it still requires changing all of the `printk()` call sites. It's also [![\[Kazuo Ito\]](https://static.lwn.net/images/conf/2010/LC-Tokyo/KazuoIto-sm.jpg)](<https://lwn.net/Articles/407535/>) possible to turn `printk()` into a macro which would assemble a message identifier from the available file name and line number information; those identifiers, though, would be too volatile for the intended use. So the approach which the developers favored was hooking into `printk()` itself to add message identifiers to messages as they find their way to the console and the logs. 

These message identifiers (also called "message-locating helper tokens") must be assigned in some sort of automatic manner; asking the development community to maintain a list of identifiers and attach them to messages seems like a sure road to disappointment. So one must immediately think of how those identifiers will be generated; the two main concerns are uniqueness and stability. It turns out that Ito-san is not concerned with absolute uniqueness; if, on occasion, two or three kernel messages end up with the same identifier, the administrator should still be able to sort things out without a great deal of pain. 

Stability is important, though; if message identifiers change frequently between releases - not to mention between boots - their value will be reduced. For that reason, generating messages at compile time using preprocessor variables like `__FILE__` and `__LINE__` to generate the identifiers, while easy, is not sufficient. One could also use the virtual address of the `printk()` call site, which is guaranteed to be unique, but that could even change from one system boot to the next, depending on things like the order in which modules are loaded. So a different approach needs to be found. 

What this group has settled on is generating a CRC32 hash of the message format string at run time. There is a certain runtime cost to that which would have been nice to avoid, but it's not that high and, if `printk()` calls are a bottleneck to system performance, there are other problems. If the system has been configured to output message identifiers, this hash value will be prepended (with a "`(%08x):`" format) to the message before it is printed. A CRC32 hash is not guaranteed to produce a unique identifier for each message (though it is better than CRC16, which is guaranteed to have collisions with 75,000 messages), but it will be close enough. 

Discussion of the current implementation during the talk revealed that there are some remaining problems. Messages printed with `dev_printk()` will all end up with the same identifier, which is an undesirable result. The newly-added "`%pV`" format directive (which indicates the passing of a structure containing a new format string and argument list) also complicates things significantly by adding recursive format string processing. So the implementation will require some work, but there was not a lot of disagreement over the basic approach. 

It was only toward the end of the talk that there was some discussion of what the use cases for this feature are. The initial goal is simply to make it easier to find where a message is coming from in the kernel code. The use of macros, helper functions, etc. can make it hard to track down a message with a simple `grep` operation. But, with a message ID and a supporting database (to be maintained with a user-space tool), developers should be able to go directly to the correct `printk()` call. Vinod Kutty noted that, in large installations, automatic monitoring systems could use the identifiers to recognize situations requiring some sort of response. There are also long-term goals around creating databases of messages translated to other languages and help information for specific messages. 

So there are real motivations for this sort of work. But, as was noted back at the beginning, getting any kind of message identifier patch through the process has always been a losing proposition so far. It is hoped that, this time around, the solution will be sufficiently useful (even to kernel developers) and sufficiently nonintrusive that it might just get over the line. We should find out soon; once the patch has been fixed, it will be posted to the mailing list for comments.
