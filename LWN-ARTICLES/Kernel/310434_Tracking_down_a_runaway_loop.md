---
title: "Tracking down a \"runaway loop\""
url: https://lwn.net/Articles/310434/
date: "December 10, 2008"
category: Bootstrap process; Modules
author: "By Jake Edge December 10, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
December 10, 2008

The Linux boot process, at least as provided by distributions, depends on help from user space, with drivers being loaded as required from the initial filesystem (initramfs/initrd). Loading drivers requires using tools built into initramfs and if those tools break, the kernel won't boot. But when a working kernel configuration and initramfs are used with a new kernel, the result is expected to be a kernel that successfully boots. When that doesn't happen, bugs are filed regarding kernel regressions but, as a recent example shows, the actual problem may be elsewhere. 

The original [report](<https://lwn.net/Articles/310472/>) was made in late October, but no progress was made until Evgeniy Polyakov [saw it again](<https://lwn.net/Articles/310473/>) in early December. The symptom was a kernel that hangs after printing: 

```
request_module: runaway loop modprobe char-major-5-1
```

four times on the console. Since nothing in the user space (initramfs) or kernel configuration had changed, it seemed to clearly point to something in the kernel itself. 

It turns out that the "runaway loop" message is meant to indicate that the `request_module()` function has been invoked recursively. So in an effort to load the driver for the character device with major/minor numbers 5/1--which corresponds to `/dev/console`--`request_module()` was invoked again. The code in `kernel/kmod.c`: 

```
if (atomic_read(&kmod_concurrent) > max_modprobes) {
                    /* We may be blaming an innocent here, but unlikely */
                    if (kmod_loop_msg++ < 5)
                            printk(KERN_ERR
                                   "request_module: runaway loop modprobe %s\n",
                                   module_name);
                    atomic_dec(&kmod_concurrent);
                    return -ENOMEM;
            }
```

only allows that message to be printed four times, but the invoker should recognize the `ENOMEM` and handle it appropriately. 

The root cause was that something in the kernel was trying to access `/dev/console` before that device was registered in the kernel. This led the kernel to try and load a module to handle `/dev/console`, which will fail. Because of the failure, something in the user space `modprobe` then tries to access `/dev/console`, presumably to output an error message, which repeats the kernel module loading process. And so on. After that recurses enough to exceed the `max_modprobes` limit, `request_module()` will produce the runaway loop message and return `ENOMEM` which _should_ put a stop to the whole process. 

In an acrimonious thread--and [kernel bug report](<http://bugzilla.kernel.org/show_bug.cgi?id=12153>)--Alan Cox, Kay Sievers, and Polyakov tried to determine where the problem came from and what to do about it. It didn't help matters that they were using different distribution's initramfs so that they saw different behavior. Polyakov/Sievers were using Debian user space while Cox was using Fedora. Something in the Debian version was continuing to try to open `/dev/console` even after getting `ENOMEM`. This leads to an infinite loop, thus a kernel hang. 

Sievers eventually [tracked it to the kernel cryptographic API](<https://lwn.net/Articles/310489/>): 

It is caused by: "modprobe cryptomgr" called from swapper[1] 

This modprobe process does try to log an error, accesses /dev/console, which is not initialized in the kernel at that time, and the kernel module loader tries the load a module to support dev_t 5:1, which again runs modprobe, and ... 

Setting CONFIG_CRYPTO_MANAGER=y makes it disappear. 

It turns out that the crypto layer attempts to load the cryptomgr module as part of its algorithm testing infrastructure. If cryptomgr fails to load, the algorithm registration code can continue without it. It is optional, but `modprobe` wants to put out a message when it fails to load it, which leads to the runaway loop. As Herbert Xu [points out](<https://lwn.net/Articles/310492/>), though, the problem is not crypto-specific at all: 

In any case the loop itself does not involve any crypto components so I don't think making changes in the crypto layer is going to make this go away forever as anyone calling request_module early enough will get into this loop. 

It is this potential pitfall that Sievers and Polyakov would like to see removed. In general, user-space programs are not required to be concerned with the availability of `/dev/console`--except when they are run from early kernel initialization. But Cox points out that user-space helpers must concern themselves with avoiding loops because there are multiple possible ways to cause that to happen. As an example, he notes that if UNIX-domain sockets (AF_UNIX) are in a module and `syslog()` is called before the module is loaded, a similar loop will occur. 

In an effort to "step back" from the arguments that were going back and forth, Ted Ts'o [offers his analysis](<https://lwn.net/Articles/310523/>) of the problem along with a suggested course of action: 

There is a dispute about whether it is looping forever, or whether it should be getting caught by kernel/kmod.c's modprobe recursion detector. Alan has checked the recursion detector and reports that it works just fine; Evgeniy and Kay are claiming that it in fact loops forever, and the recursion detector is not working. 

[...] So I would think the best thing to do is to figure out what Debian's initrd is doing that is evading the recursion detection. Fixing that is going to make things much more robust. 

Clearly the recursion detector is working to some extent, or the runaway loop messages would not be seen, but on Debian, at least, that detection doesn't stop the problem. Ts'o's theory is that something outside of directly invoked helper is actually the culprit: ""I'm guessing why it isn't working given Debian's initrd setup is that whatever is ultimately opening /dev/console isn't being called until after the helper script has exited."" That seems worth tracking down as Ts'o points out in a [later message](<https://lwn.net/Articles/310524/>): 

It would be good to make sure we understand what the root causes for while the modprobe recursion detector is apparently not triggering, since it could be that Debian's initrd might cause some other uncaught recursion loop if we don't drive this problem determination to root cause. 

The exact cause of the problem and why Debian and Fedora behave differently is still not known. Digging into Debian's initrd to figure that out, as Ts'o suggests, is clearly the right starting point. That answer will likely lead to sensible fixes, either in user space or the kernel--possibly both. Bickering about where and how to fix the problem before it is fully understood seems counter-productive at best.
