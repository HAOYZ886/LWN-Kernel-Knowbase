---
title: Finding buffer overflows with Parfait
url: https://lwn.net/Articles/344003/
date: "July 29, 2009"
category: Development tools
author: "By Jake Edge July 29, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
July 29, 2009

Recently, Roel Kluin has been proposing patches to fix a number of buffer overflows in the kernel, some of which he credited to "Parfait". It turns out that [Parfait](<http://research.sun.com/projects/parfait/>) is a static source code checking tool that comes out of Sun Labs in Australia. The project reported 54 buffer overflows to the linux-security mailing list in early July, and Kluin has been going through them to get them fixed. 

It is best to treat buffer overflows as potential security vulnerabilities, even though they may be hard--or impossible--to exploit. Various types of these bugs have been thought to be unexploitable along the way, but then were found to be exploitable, so caution is clearly indicated. The full list was sent to the private kernel security alias, and then passed along to Kluin by Andrew Morton. Kluin has then been posting patches to linux-kernel, as well as the netdev mailing list, to fix them. A number of the fixes have already been picked up by subsystem maintainers, and some have made their way into the mainline. 

The tool itself is relatively new, first demonstrated as an alpha last October, and takes a multi-layered approach using an "ensemble" of static analysis techniques. Thus the name. One of the goals, from the outset, was to produce something that could analyze a huge codebase--the OpenSolaris or Linux kernel for example--in a matter of minutes rather than the days or weeks that other tools require. 

As part of a [paper [PDF]](<http://research.sun.com/projects/downunder/publications/documents/kca09.pdf>) presented at the Kernel Conference Australia in mid-July, the Parfait developers reported checking 5.7 million lines of code in the 2.6.29 kernel for buffer overflows in 13 minutes. The times for OpenSolaris and OpenBSD were similar when scaled for the number of lines of code checked. 

Unsurprisingly, for all three kernels, the majority of buffer overflows were found in the driver code. For 2.6.29, Parfait found 12 buffer overflows in the Linux core, and 85 in the drivers (which makes up 71% of the codebase). Some of those were false positives, but the paper does not make it clear just how many. Given that 54 were reported to linux-security, though, it would seem that something approaching half were false positives. 

Kluin provided an example of the Parfait output: 

```
Bug type: Buffer overflow
        File: /usr/src/linux-2.6.29/security/smack/smackfs.c
        Line: 777
        Function: smk_write_netlbladdr
        Code snippet:
    
        0772:   if (count < SMK_NETLBLADDRMIN || count > SMK_NETLBLADDRMAX)
        0773:           return -EINVAL;
        0774:   if (copy_from_user(data, buf, count) != 0)
        0775:           return -EFAULT;
        0776:
        0777:   data[count] = '\0';
        0778:
        0779:   rc = sscanf(data, "%hhd.%hhd.%hhd.%hhd/%d %s",
        0780:           &host[0], &host[1], &host[2], &host[3], &m, smack);
        0781:   if (rc != 6) {
        0782:           rc = sscanf(data, "%hhd.%hhd.%hhd.%hhd %s",
    
        Parfait report:
        Error: Buffer overflow at
        /usr/src/linux-2.6.29/security/smack/smackfs.c:777 in function
        'smk_write_netlbladdr' [Symbolic analysis]
    	  In array dereference of data[count] with index 'count'
    	  Array size is 42 bytes, count >= 9 and count <= 42
    
        Comments:
        Off-by-one when adding the trailing null on line 777 - data is
        declared with size
        SMK_NETLBLADDRMAX, and count is allowed to equal SMK_NETLBLADDRMAX
```

Which shows a buffer overflow that he had already fixed in the kernel prior to the Parfait report. The paper also describes a GUI tool that collects up the code and declarations that make Parfait believe there is a problem, which can help developers determine whether there truly is a problem or not. 

Currently, Parfait is not available to those outside of Sun, but a binary release is planned. According to lead developer Cristina Cifuentes, it should be available on the web site within the next month or two: ""I estimate it will be end of August (to be optimistic) before the binary release is out, a more pessimistic estimate is end of September."" That release will be available for ""use on a non-commercial basis"", she said. Sun is considering an open source release, but no decision on that has yet been made. 

In an [interview](<http://research.sun.com/minds/2009-0512/>) on the Sun Labs web site, Cifuentes gives a broader view of where Parfait is headed--more than just looking for buffer overflows: 

At the moment the types of bugs we're finding include other memory-pointer related bugs. Things like null pointer dereference, double free, use after free, memory leaks, format string type mismatches -- they can all be found with similar types of analysis. Those are some that we're putting our emphasis on now. 

In many ways, Parfait is similar to the [Coverity](<http://www.coverity.com/>) analysis tool that has been used on the kernel as well as other free software. In both cases, at least for now, the analysis can only be run by the company who owns the tool, or those who have licensed it in the case of Coverity. A free software analysis tool that did these kinds of checks--and didn't depend on the goodwill of various companies--would be a real boon. With luck, perhaps Parfait will some day fill that role. 

These source analysis tools clearly find real bugs, though there is some [evidence](<http://www.usenix.org/events/usenix09/tech/full_papers/guo/guo_html/index.html>) that the bug reports resulting from the scans are not being used to their fullest. The Coverity scanner [found](<http://blog.coverity.com/posts/general/would-you-like-to-know-about-0day-defects-months-in-advance>) the `tun.c` [NULL pointer dereference](<http://lwn.net/Articles/342330/>) problem long before it was fixed in the kernel, but the report either went unnoticed or was (incorrectly as it turns out) not seen to be a serious problem. More source code analysis--at least any that isn't plagued by too many false positives--can only be a good thing, but the problems found need to be addressed or the value of the effort drops dramatically. It would be awfully nice to have free versions of these kinds of tools as well.
