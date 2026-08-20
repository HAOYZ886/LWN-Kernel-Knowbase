---
title: Two LSS talks
url: https://lwn.net/Articles/569861/
date: "October 9, 2013"
category: Security
author: "By Jake Edge October 9, 2013 Linux Security Summit"
---

By **Jake Edge**  
October 9, 2013

* * *

[Linux Security Summit](<https://lwn.net/Archives/ConferenceByYear/#2013-Linux_Security_Summit>)

The 2013 [Linux Security Summit](<http://kernsec.org/wiki/index.php/Linux_Security_Summit_2013>) (LSS) had many different talks over its first day. It started with Ted Ts'o's [keynote](<https://lwn.net/Articles/567977/>), then had several "refereed" talks (two of which were covered here: [Embedded integrity](<https://lwn.net/Articles/568943/>) and [Kernel ASLR](<https://lwn.net/Articles/569635/>)). The day ended with a selection of short topics, which were mostly updates on various security subsystems (SELinux, Smack, and Integrity). Unfortunately, there isn't enough time to write all of them up, but we will complete our LSS coverage with reports from two kernel-related talks. 

#### LSM composition

Composing (or stacking) Linux Security Modules (LSMs) has been a perpetual topic in kernel security circles. We have looked at the problem and solutions many times over the years (most recently [in August](<https://lwn.net/Articles/563178/>)). Casey Schaufler has been working on a solution to the problem over the last few years. He presented the problem and his solution to the assembled security developers looking for guidance on where to go from here. 

The problem in a nutshell is that there can be only one LSM active on any given boot of the kernel. But there are multiple use cases for having more than one active. Many think that certain combinations (e.g. SELinux and Smack) do not make sense, but there are reports of people who want to be able to combine arbitrary LSMs. In addition, only allowing a single LSM restricts systems with multiple containers to only using one security model, when it might be desirable to support all of them for use in different containers. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There is also the problem of special-purpose LSMs. Whenever someone brings up an idea for a new way to tighten security in the kernel core, they are almost inevitably pointed in the direction of the LSM API. But, since many distributions already ship with the single LSM slot filled, those smaller LSMs are unlikely to be used. Yama started off as a special-purpose LSM, but it was eventually manually—unconditionally—stacked so that it could coexist. That was meant as a temporary change until stacking was added to the kernel, but without a stacking solution, its temporary nature is somewhat in question. 

Schaufler's proposal still follows the broad outlines we [described](<https://lwn.net/Articles/518345/>) a year ago. It has added the ability to stack any and all of the existing LSMs, which was not working at that point. It has also added a user-space interface that has separate directories for each LSM under `/proc/PID/attr`. It now tries to deal with labeled networking by restricting the different mechanisms (NetLabel, XFRM, and secmark) each to a single LSM per boot. The first LSM that asks for a given network labeling scheme "gets it". The details are available in Schaufler's [slides [PDF]](<http://selinuxproject.org/~jmorris/lss2013_slides/casey_Glass2013.pdf>) as well as the [patches](<https://lwn.net/Articles/560802/>). But the point of his talk was mostly to get feedback and ideas on whether it was an idea worth moving forward with. 

Some were not particularly happy with the user-space interface and/or the networking changes, believing that they added too much complexity. Others seemed skeptical that stacking was ever a sensible thing to do. But several folks spoke up from the audience about how they currently use multiple LSMs and have to carry out-of-tree patches to make it all work. In addition, the standard stacking arguments were restated. There _is_ a clear demand for the feature—whether that is enough to overcome the objections remains to be seen. 

In a post-presentation discussion, Paul Moore and Schaufler explored the possibility of pushing forward the stacking core, while leaving some of the "messier" pieces (like the user-space interface and labeled networking handling) as later additions. Most or all of the stated use cases would be fully served by a "simplified stacking" solution. The other pieces could continue to be worked on, or possibly dropped if there were no real users for them. That sounded like the approach that will be tried next, but, so far, patches have not yet appeared. 

#### Core kernel anti-patterns

There are lots of known "anti-patterns" for kernel code, like busy waiting or hardcoding values, but _security_ anti-patterns are not as well-known, Kees Cook said at the beginning of his talk. He and others have been spending some time trying to find "obvious" bugs in the kernel, some of which fall into the anti-pattern category. His talk was meant to document some of them to hopefully avoid them in the future. 

It is frustrating that he can so easily find security holes in the kernel, he said. In addition, Dan Carpenter has been using [smatch](<http://smatch.sourceforge.net/>) to find more examples of these anti-patterns once someone has found the first instance. Cook suggested that perhaps `checkpatch.pl` could be extended to catch some of this bad code long before it ever reached the kernel. He also suggested that kernel developers just go look for other examples of "something ugly" when they find such bugs—surprisingly often they will find many more instances. 

Format strings are one source of these errors. For example: 

```
printk(buffer);
```

If the user can influence what goes into the buffer, they can put format specifiers into it, which can cause havoc. Fixing the problem is as easy as: 

```
printk("%s", buffer);
```

GCC can be used to help find these kinds of problems, using the `format` and `format-security` warning/error flags, but it is, unfortunately, "dumb" about `const char *`, so it will complain even for static buffers that are not exploitable. 

A related problem is the addition of the "`%n`" format specifier, which writes the number of characters written to an address that is passed as an argument on the stack. It was not added to the kernel until 2009 and is only used for padding calculations in procfs output. But it is the format specifier of choice for those performing format string attacks. He would like to see support for that specifier removed entirely: "I don't care about prettiness if it leaves `%n` as an attack vector." 

String manipulation is another area with lots of low-hanging fruit. He noted that `strncpy()` is generally a safer call than some others (e.g. `strcpy()`), but you have to check the length of the destination, not the source. 

```
strncpy(dest, src, strlen(src));
```

can sometimes be found and it will leave the `dest` string without a NULL termination. He suggested that for purposely leaving the destination unterminated, one should use `memcpy()` to make it clear. 

Another problem that is fairly easy to find is unchecked `copy_*_user()` calls. The return from those is the number of bytes not copied, which typically indicates some kind of error. So calling those routines without checking the return value can lead to security holes. Various graphics drivers are currently guilty, he said. 

Reading from the same user-space location twice can lead to a race condition where the value changes between the two reads. It is a hard race to win, but still a problem. This often happens when the first part of a structure being read from user space is the length of the data. The length is read in, the structure is allocated, then the whole thing (length and all) is read into the new structure. If the length changes between the reads, it can lead to problems. He has found this anti-pattern in both the kernel and U-Boot. 

A problem similar to the double-read occurs in drivers for unusual devices. USB human interface devices (HID) have a complex way of describing the data being delivered. In a week's time, he found 12 CVEs in that code using malicious hardware. He verified each using a [Facedancer](<http://travisgoodspeed.blogspot.com/2012/07/emulating-usb-devices-with-python.html>) a software-defined USB device, which allows him to write a Python script that acts like a USB device. In the future, he plans to look for problems in the mass storage and webcam USB drivers. 

Cook said these kinds of bugs are an indication that the "many eyes" theory is failing in some areas. He knows this because he keeps finding the same kinds of bugs whenever he has time to look. There are tools that could help, including stronger GCC default flags and using the GCC plugins from the PaX project. Coccinelle and smatch are also useful. It is important that we get more proactive, he said, and keep these anti-patterns from ever getting into the kernel to begin with. 

[I would like to thank LWN subscribers for travel assistance to New Orleans for LSS.]
