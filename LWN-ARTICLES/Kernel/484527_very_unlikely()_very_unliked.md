---
title: very_unlikely() very unliked
url: https://lwn.net/Articles/484527/
date: "February 29, 2012"
category: Jump label; Static keys
author: "By Jonathan Corbet February 29, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 29, 2012

The facility formerly known as [jump labels](<https://lwn.net/Articles/436041/>) is designed to minimize the cost of rarely-used features in the kernel. The classic example is tracepoints, which are turned off almost all of the time. Ideally, the overhead of a disabled tracepoint would be zero, or something very close to it. Jump labels (now "static keys") use runtime code modification to patch unused features out of the code path entirely until somebody turns them on. 

With Ingo Molnar's encouragement, Jason Baron recently posted [a patch set](<https://lwn.net/Articles/482817/>) changing the jump label mechanism to look like this: 

```
if (very_unlikely(&jump_label_key)) {
    	/* Rarely-used code appears here */
        }
```

A number of reviewers complained, saying that `very_unlikely()` looks like just a stronger form of the `unlikely()` macro, which works in a different way. Ingo [responded](<https://lwn.net/Articles/484528/>) that the resemblance was not coincidental and made sense, but he made few converts. After arguing the point for a while, Ingo [gave in](<https://lwn.net/Articles/484529/>) to the inevitable and stopped pushing for the `very_unlikely()` name. 

Instead, we have yet another format for jump labels, as described in [this document](<https://lwn.net/Articles/484530/>). A ~~jump label~~ static key is defined with one of: 

```
struct static_key key = STATIC_KEY_INIT_FALSE;
        struct static_key key = STATIC_KEY_INIT_TRUE;
```

The initial value should match the normal operational setting - the one that should be fast. Testing of static keys is done with: 

```
if (static_key_false(&key)) {
    	/* Unlikely code here */
        }
```

Note that `static_key_false()` can only be used with a key initialized with `STATIC_KEY_INIT_FALSE`; for default-true keys, `static_key_true()` must be used instead. To make a key true, pass it to `static_key_slow_inc()`; removing a reference to a key (and possibly making it false) is done with `static_key_slow_dec()`. 

This version of the change was rather less controversial and, presumably, will find its way in through the 3.4 merge window. After that, one assumes, this mechanism will not be reworked again for at least a development cycle or two.
