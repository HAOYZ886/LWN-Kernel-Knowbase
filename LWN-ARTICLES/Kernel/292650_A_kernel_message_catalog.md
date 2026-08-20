---
title: A kernel message catalog
url: https://lwn.net/Articles/292650/
date: "August 4, 2008"
category: Messages
author: "By Jonathan Corbet August 4, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 4, 2008

Kernel developers will often use `printk()` to output a message when something goes wrong. Such messages tend to be helpful to kernel developers; if nothing else, they can be used to find the place in the source where the message is emitted, and that, in turn, is most useful for somebody trying to figure out what the message is really saying. So, if your kernel tells you, for example, "lguest is afraid of being a guest," a quick dig through the source turns up a comment reading "Lguest can't run under Xen, VMI or itself. It does Tricky Stuff." Problem solved - or, at least, understood. 

But, for the bulk of Linux users and administrators, the act of `printk()` interpretation by recourse to the kernel source is, itself, Tricky Stuff. If the kernel cannot tell them directly what the problem is, they would much rather have a more straightforward means of translating messages into some sort of useful English. 

Or maybe not: for many Linux users, English may not be much more helpful than straight kernel-speak. It would be really nice to translate those messages into some sort of useful French, or Chinese, etc. What it comes down to, in the end, is that `printk()` alone will never be able to provide sufficient information to users in a way which can be understood and used to solve problems. 

Just over one year ago, LWN [looked at some proposals](<http://lwn.net/Articles/238948/>) for adding structure to kernel messages. After that, the discussion went quiet, to the point that it seemed like not much was happening in the messaging area. But one should not forget that we are dealing with companies like IBM which have been creating massive binders full of kernel message documentation for several decades. They're not going to give up so easily. So the posting (by Martin Schwidefsky) of [a new kernel messaging proposal](<http://lwn.net/Articles/292560/>) is not an entirely surprising event. 

In the latest scheme, each source file which generates structured messages defines a macro `KMSG_COMPONENT` as a string naming the specific subsection. This name will often match the name of the module which is created from that code, but that is not necessarily the case. The name, once chosen, is supposed to remain fixed forevermore; it becomes, in essence, part of the user-space interface and should always match the documentation. 

Then, each message is assigned an integer identification number. The combination of the component name and the message number should be unique throughout the kernel; it is used by various tools to associate a more detailed explanation of whatever the message is intended to communicate. The message number is used with one of a number of new `printk()`-like functions: 

```
kmsg_alert(id, format, args...);
        kmsg_err(id, format, args...);
        kmsg_warn(id, format, args...);
        kmsg_info(id, format, args...);
        kmsg_notice(id, format, args...);
    
        kmsg_dev_alert(id, dev, format, args...);
        /* ... */
```

The "`_dev`" versions take an additional `struct device` argument (like `dev_printk()`) and encode the device name in the resulting message. That message (for all variants) will include the component name and the message number in any output. So, for example, the S/390 "xpram" driver includes the following: 

```
#define KMSG_COMPONENT "xpram"
    
            /* ... */
            if (devs <= 0 || devs > XPRAM_MAX_DEVS) {
    	    kmsg_err(1, "%d is not a valid number of XPRAM devices\n", devs);
```

Should this particular error check trigger, the resulting message will look like this: 

```
xpram.1: 42 is not a valid number of XPRAM devices
```

Thus far, our user is probably not feeling much better informed than before. But there is additional information which is made available and associated with that message tag. In this particular case, it looks like this: 

```
/*?
     * Tag: xpram.1
     * Text: "%d is not a valid number of XPRAM devices"
     * Severity: Error
     * Parameter:
     *   @1: number of partitions
     * Description:
     * The number of XPRAM partitions specified for the 'devs' module parameter
     * or with the 'xpram.parts' kernel parameter must be an integer in the
     * range 1 to 32. The XPRAM device driver created a maximum of 32 partitions
     * that are probably not configured as intended.
     * User action:
     * If the XPRAM device driver has been compiled as a separate module,
     * unload the module and load it again with a correct value for the
     * 'devs' module parameter. If the XPRAM device driver has been compiled
     * into the kernel, correct the 'xpram.parts' parameter in the kernel
     * parameter line and restart Linux.
     */
```

Here, we have a more verbose description of the message. Even more helpfully (one hopes), there is a discussion of what can be done to make this message go away. This information can be provided within the source or in a separate documentation file; it can also, presumably, be nicely formatted and distributed to paying customers as a binder for the system administrator's bookshelf. It can be translated into other languages for Linux users worldwide (and beyond: one could have a lot of fun with the Klingon translation for this kind of material). 

The patch includes a script (written in Perl with undocumented messages, of course) which (when invoked with `make D=1`) will go through the source and make sure that every kernel message has an associated description block; it can also format the descriptions into man pages if desired. There are checks for missing descriptions or overloaded message ID numbers; the script does not, at the moment, check for a change in the message text. 

Martin's first posting made this work specific to the S/390 architecture; following [a suggestion from Andrew Morton](<https://lwn.net/Articles/292660/>), he made it generic in later versions. The cost of this work is zero for those who do not use it, so there is a reasonable chance that it will find its way into the mainline eventually. Before the message catalog system can be truly useful, though, developers will have to go through and document a substantial portion of the messages created by the kernel - and keep that documentation current as the kernel evolves.
