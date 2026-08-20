---
title: Being honest with MODULE_LICENSE
url: https://lwn.net/Articles/82305/
date: "April 27, 2004"
category: "Modules-Licensing; Tainting"
author: 
---

`MODULE_LICENSE()` is a macro which allows loadable kernel modules to declare their license to the world. Its purpose is to let the kernel developers know when a non-free module has been inserted into a given kernel. If you submit an oops report showing a "tainted" kernel, chances you will be asked to reproduce the problem without the proprietary module loaded, or to talk to that module's vendor about the problem. In general, the kernel hackers want to hear about problems, but their interest drops remarkably when they cannot get at the source to diagnose or fix the problem. 

The declared module license is also used to decide whether a given module can have access to the small number of "GPL-only" symbols in the kernel. 

There is no central authority which checks license declarations; it is assumed that module authors will not want to lie about the license they are using. That assumption has generally proved to be valid, so people were surprised when Linuxant [was found](<https://lwn.net/Articles/82306/>) to have put a false module declaration into its binary-only "linmodem" driver. Or, if it's not false, it does cleverly manage to not tell the whole story. 

The actual license string in the Linuxant driver reads: 

GPL\0for files in the "GPL" directory; for others, only LICENSE file applies 

The `\0` is an ASCII NUL character, which, in C programs, terminates a string. Thus, while the above declaration would appear fairly clear to human eyes, the kernel only sees a license declaration of "GPL". 

One might well wonder why Linuxant chose to do this. The driver in question does not use any GPL-only symbols, so it is not an attempt to get around the kernel's simplistic access control mechanism. [According to Linuxant president Marc Boucher](<https://lwn.net/Articles/82309/>), they simply wanted to avoid bothering users with kernel warnings: 

The purpose of the workaround is to avoid repetitive warning messages generated when multiple modules belonging to a single logical "driver" are loaded (even when a module is only probed but not used due to the hardware not being present). Although the issue may sound trivial/harmless to people on the lkml, it was a frequent cause of confusion for the average person. 

Most developers seem to have taken this explanation at face value, though some [remain unhappy](<https://lwn.net/Articles/82452/>) about the approach that was used. Possible solutions include putting the "kernel tainted" warning in the system logfile only, simply suppressing the warning after the first time, or having the Linuxant drivers manually set the "tainted" flag themselves at load time. Finding a way to achieve Linuxant's aim (provide a driver which enables hardware that does not otherwise work with Linux while avoiding upsetting users with lots of scary messages) should not be that hard to do. 

Meanwhile, of course, there is also interest in making it harder for others to get past the kernel license check. Carl-Daniel Hailfinger, who originally pointed out the problem, also submitted a patch which would explicitly "blacklist" modules from Linuxant; any such module would taint the kernel regardless of its claimed license. Linus [suggested](<https://lwn.net/Articles/82313/>) that the license be stored as a counted string as a way of defeating the "NUL attack." Rusty Russell, instead, [noted](<https://lwn.net/Articles/82314/>) that any check that would be accepted into the kernel can be defeated by an even moderately motivated attacker. His patch includes a quick compile-time check to defeat Linuxant's technique, but it explicitly avoids getting into a real arms race with potential violators. 

Chances are we will see this sort of behavior again - with, perhaps, a less benign intent. The nature of a free kernel makes it hard to shut out those who are unwilling to play by the rules. But, as Linus said: 

...playing the above kinds of games makes it pretty clear to everybody that any infringement was done wilfully. They should be talking to their lawyers about things like that. 

Given that a number of free software hackers are increasingly unwilling to see their licenses ignored, anybody who wants to engage in this sort of behavior should, indeed, be talking to their lawyers.
