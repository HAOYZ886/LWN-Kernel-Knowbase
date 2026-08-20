---
title: Toward more reliable logging
url: https://lwn.net/Articles/492125/
date: "April 13, 2012"
category: Kernel messages; Messages
author: "By Jonathan Corbet April 13, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 13, 2012

Messages from the kernel are created by humans, usually using one of the many variants of the `printk()` function. But, increasingly, those messages are read by machines in the form of log file parsers, automated management systems, and so on. The machines have, for some time, struggled to make sense out of those human-created messages which, often as not, are unpredictable in their organization, lacking important information, and subject to change. So it is not surprising that there has been ongoing interest in adding some structure to kernel log messages; the subject was recently raised by the audience at the Collaboration Summit [kernel panel](<https://lwn.net/Articles/491258/>). Even so, almost every attempt to improve kernel logging has failed to make much (if any) headway. 

The same fate seemed to be in store for Lennart Poettering and Kay Sievers when they presented their ideas at the [2011 Kernel Summit](<https://lwn.net/Articles/464276/>); in particular, their concept of attaching 128-bit unique ID numbers to each message was met with almost universal disdain. Lennart and Kay have not given up, though. The latest form of their work on the kernel side of the problem can be seen in the [structured `printk()` patch](<https://lwn.net/Articles/490690/>) recently posted by Kay. 

The patch does a few independent things - a cause for a bit of complaining on the mailing list. The first of these is to change the kernel's internal log buffer from a stream of characters into a series of records. Each message is stored into the buffer with a header containing its length, sequence number, facility number, and priority. In the end, Kay says, the space consumed by messages does not grow; indeed, it may shrink a bit. 

The record-oriented internal format has a number of advantages. If messages are being created faster than they can be consumed by user space, it is necessary to overwrite the oldest ones. Current kernels simply write over the character stream, with the result that truncated messages can find their way into the log. The new implementation drops entire messages at once, so the stream, while no longer complete, is not corrupted. The sequence numbers allow any consumer to know that messages have been dropped \- and exactly how many it missed. The record-oriented format also enables improved handling of continuation lines at the cost of making the use of the `KERN_CONT` "priority" mandatory for such lines. 

The second change is to allow the addition of a facility number and a "dictionary" containing additional information that, most likely, will be of interest to automated parsers. The dictionary contains "`KEY=value`" pairs, separated by spaces; these pairs will contain, for example, device and subsystem names to unambiguously identify the device that generated the message. Kernel code that wants to attach a facility number and/or dictionary to a message will use the new function `printk_emit()` to do so: 

```
int printk_emit(int facility, int level, const char *dict, 
    			size_t dictlen, const char *fmt, ...);
```

Regular `printk()` turns into a call to `printk_emit()` with a `facility` of zero and a null `dict`. 

Creation of the dictionary string itself is left as an exercise for the caller; it is not something one is likely to see done in most places where `printk()` is called. In fact, the only full user of `printk_emit()` in the patch is `dev_printk()` which uses it to add a dictionary with `SUBSYSTEM` and `DEVICE` fields describing the device of interest. If some form of this patch is merged, one can expect this usage pattern to continue; the creation of dictionaries with ancillary information will mostly be done with subsystem-specific print functions. 

Finally, the patch changes the appearance of log messages when they reach user space. After [some complaints from Linus](<https://lwn.net/Articles/492134/>), the format has evolved to look something like this: 

```
7,384,6193747;sr 1:0:0:0: Attached scsi CD-ROM sr0
         SUBSYSTEM=scsi
         DEVICE=+scsi:1:0:0:0
```

The comma-separated fields at the beginning are the message priority (7, in this case), the sequence number (384), and the system time in microseconds since boot. The rest of the line is the message as passed to `printk()`. If the message includes a dictionary, it appears in the following lines; following the style set in RFC 821, continuation lines begin with white space. The result, it is hoped, is an output format that is simultaneously easy for humans to read and machines to parse. 

The behavior of the `/dev/kmsg` device changes somewhat as well. In current kernels, this device is only useful for injecting messages into the kernel log stream. Kay's patch turns it into a device supporting `read()` and `poll()` as well, with multiple concurrent readers supported. If messages are overwritten before a particular reader is able to consume them, the next `read()` call will return an `EPIPE` error; subsequent reads will start from the next available message. This device thus becomes a source of kernel log data that is easy to work with and that reliably delivers log messages or ensures that the recipient knows something was lost. 

Modulo some quibbling over the log format, the response to the patch seems to be mostly positive. The biggest exception is arguably Ingo Molnar, whose [suggestion](<https://lwn.net/Articles/492273/>) that tracepoints and perf should be used instead does not appear to have received a lot of support. Even Linus seems mostly happy; the absence of the 128-bit unique ID perhaps has a lot to do with that. But, beyond that, a more robust log buffer with sequence numbers has some clear advantages; Linus [suggested](<https://lwn.net/Articles/492140/>) that, if that part were split out, he might even consider merging it for the 3.4 release. That seems unlikely to happen at this point in the cycle, but it wouldn't be surprising to see something come together for the 3.5 development cycle. If that happens, Linux will still lack a true structured logging mechanism, but it would have something with more structure and reliability than it has now.
