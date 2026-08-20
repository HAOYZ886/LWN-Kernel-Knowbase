---
title: Jump label reworked
url: https://lwn.net/Articles/436041/
date: "March 30, 2011"
category: Jump label
author: "By Jonathan Corbet March 30, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 30, 2011

The [jump label](<https://lwn.net/Articles/412072/>) mechanism was last seen here in October, 2010. In short, jump label allows the optimization of "highly unlikely" code branches to the point that their normal overhead is close to zero. This speedup is done with runtime code patching; that is also the cost: enabling or disabling the unlikely case is an expensive operation. Thus, jump label is best used for code which is almost never enabled; tracepoints and [dynamic debugging](<https://lwn.net/Articles/434833/>) statements are obvious cases. 

There were a number of complaints about the initial jump label implementation, including the fact that it was somewhat awkward to use. In response, [a reworked version](<https://lwn.net/Articles/436043/>) has been posted which changes the interface considerably. One starts by declaring a "jump key": 

```
#include <linux/jump_label.h>
    
        struct jump_label_key my_key;
```

Enabling and disabling the key is a simple matter of calling: 

```
jump_label_inc(struct jump_label_key *key);
        jump_label_dec(struct jump_label_key *key);
```

And using the key to control the execution of rarely-needed code becomes: 

```
if (static_branch(&my_key)) {
    	/* Unlikely stuff happens here */
        }
```

In the absence of full jump label support, a jump key is represented by an `atomic_t` value. `jump_label_inc()` becomes `atomic_inc()`, `jump_label_dec()` becomes `atomic_dec()`, and `static_branch()` is implemented with `atomic_read()`. If jump label is configured into the kernel, enabling and disabling a jump key become heavier operations, while `static_branch()` becomes nearly free. For the intended use cases for jump labels, that is a worthwhile tradeoff. 

As of this writing, these changes have not been merged for 2.6.39. There is always a possibility that they could be pulled in before -rc2, but chances are that, at this point, the new jump label will have to jump into 2.6.40.
