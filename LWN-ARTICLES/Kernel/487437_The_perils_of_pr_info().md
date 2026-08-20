---
title: The perils of pr_info()
url: https://lwn.net/Articles/487437/
date: "March 21, 2012"
category: printk
author: "By Jonathan Corbet March 21, 2012"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 21, 2012

In the beginning there was `printk()` \- literally: the 0.01 kernel release included 44 `printk()` calls. Since then, `printk()` has picked up details like logging levels and a lot of new formatting operators; it has also expanded to tens of thousands of call sites throughout the kernel. Developers often reach for it as the first way to figure out what is going on inside a misbehaving subsystem. If some developers have their way, though, `printk()` calls will become an endangered species. But not everybody has signed on to that goal. 

There are certainly plenty of ways in which `printk()` could be improved. It imposes no standardization on messages, either across a subsystem or over time. As a result, messages can be hard for programs (or people) to parse, and they can change in trivial but obnoxious ways from one kernel release to the next. The actual calls, starting with text like: 

```
printk(KERN_ERR ...
```

are relatively verbose; among other things, that often causes `printk()` statements to run afoul of the 80-column line width restriction. Messages printed with `printk()` may also lack important information needed to determine what the kernel is really trying to say. 

Various attempts have been made to improve on `printk()` over the years. Arguably the most successful of those is the set of functions defined for device drivers: 

```
int dev_dbg(struct device *dev, const char *format, ...);
        int dev_info(struct device *dev, const char *format, ...);
        int dev_notice(struct device *dev, const char *format, ...);
        /* ... */
        int dev_emerg(struct device *dev, const char *format, ...);
```

These functions, by embedding the logging level in the name itself, are more concise than the `printk()` calls they replace. They also print the name of the relevant device in standard form, ensuring that it's always possible to associate a message with the device that generated it. Use of these functions is not universal in device drivers, but it is widespread and uncontroversial. 

There is a rather lower level of consensus surrounding a different set of functions (macros, really) that look like this: 

```
int pr_info(const char *format, ...);
        /* ... */
        int pr_emerg(const char *format, ...);
```

These functions, too, encode the logging level in the function name, making things more concise. They also attempt to at least minimally standardize the format of logging by passing the `format` string through a macro called `pr_fmt()`. That leads to a line like this appearing in several hundred source files in the kernel: 

```
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
```

Due to the way the macro works, this line must appear before the `#include` block that would otherwise be at the beginning of the file. Defining `pr_fmt()` in this way causes all strings printed from the file to have the module name prepended; many subsystems use a literal string rather than the module name, but the intent is the same. 

The spread of `pr_*()` through the kernel is mainly the result of an ongoing campaign by Joe Perches - notable for having just merged [a 100,000-line whitespace-only ISDN subsystem cleanup patch](<https://git.kernel.org/linus/475be4d85a274d0961593db41cf85689db1d583c>) for 3.4 - who has converted thousands of `printk()` calls over the years. To some developers, these changes are a welcome cleaning-up of the code; to others, they represent pointless code churn. The discussion has been quiet for a while, but it recently came back when Joe [tried to convert the ext4 filesystem](<https://lwn.net/Articles/487447/>); ext4 maintainer Ted Ts'o [rejected](<https://lwn.net/Articles/487449/>) the conversion, saying: 

Changing printk's to pr_info and pr_cont is patch noise as far as I'm concerned. Adds no value, and just breaks other patches 

David Miller [commented](<https://plus.google.com/101384639386588513837/posts/KaRo6LaVCx5>) on this decision in a rather unsympathetic fashion: 

Some kernel maintainers are real blockheads about code cleanups. And being like that doesn't make you look established and sophisticated, instead it makes you look like what you actually are, a relic. 

Ted probably does not feel like a relic, and he is probably not trying to be sophisticated; he is almost certainly trying to maintain code he is responsible for in the best way he can. In his view, changing a bunch of code from one print function to another - possibly introducing a lot of patch conflicts on the way - does not help in that regard. Beyond that, he [said](<https://lwn.net/Articles/487452/>), the standardization introduced by these functions is nowhere near enough to solve the structured logging problem, meaning that, someday, all those calls will have to be changed yet another time when a proper solution is available. 

Proponents of the change argue that some structure is better than none, and that the new functions offer some useful flexibility when the time to add more structure comes. They claim that the overall size of the kernel is reduced (slightly) due to better sharing of strings. Messages printed with `pr_debug()` can be enabled and disabled with the [dynamic debugging interface](<https://lwn.net/Articles/434833/>), while straight `printk()` calls cannot. And, perhaps most of all, they argue that consistency across the code base has value - though that argument was heard rather less when the `pr_*()` interface was new and relatively unused. 

Needless to say, this is not the kind of discussion that comes to any sort of definitive conclusion. With regard to ext4, the conversion will probably not take place anytime soon; that is Ted's turf, and it is unlikely that anybody can summon arguments strong enough to convince Linus to override him. Elsewhere in the kernel, though, these conversions will certainly continue. As will, undoubtedly, the associated flame wars.
