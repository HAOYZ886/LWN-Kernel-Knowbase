---
title: Jump label
url: https://lwn.net/Articles/412072/
date: "October 27, 2010"
category: Jump label
author: "By Jonathan Corbet October 27, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 27, 2010

The kernel is filled with tests whose results almost never change. A classic example is tracepoints, which will be disabled on running systems with only very rare exceptions. There has long been interest in optimizing the tests done in such places; with 2.6.37, the "jump label" feature will make those tests go away entirely. 

Consider the definition of a typical tracepoint, which, behind all of the preprocessor madness, looks something like: 

```
static inline trace_foo(args)
        {
      	if (unlikely(trace_foo_enabled))
    	    goto do_trace;
            return;
        do_trace:
    	/* Actually do tracing stuff */
        }
```

The cost of a test for a single tracepoint is essentially zero. The number of tracepoints in the kernel is growing, though, and each one adds a new test. Each test must fetch a value from memory, adding to the pressure on the cache and hurting performance. Given that the value almost never changes, it would be nice to find a way to optimize the "tracepoint disabled" case. 

In 2.6.37, this tracepoint can be rewritten using a new macro: 

```
#include <linux/jump_label.h>
    
        #define JUMP_LABEL(key, label)		\
    	if (unlikely(*key))			\
    		goto label;
```

The nice thing is that `JUMP_LABEL()` does not have to be implemented like that. It can, instead, (1) note the location of the test and the `key` value in a special table, and (2) simply insert a no-op instruction. That reduces the cost of the test (and the tracepoint) to zero for the common "not enabled" case. Most of the time, the tracepoint will never be enabled and the omitted test will never be missed. 

The tricky part happens when somebody wants to enable the tracepoint. Changing its status now requires calling one of a pair of special functions: 

```
void enable_jump_label(void *key);
        void disable_jump_label(void *key);
```

A call to `enable_jump_label()` will look up the key in the jump label table, then replace the special no-op instructions with the assembly equivalent of "`goto label`", enabling the tracepoint. Disabling the jump label will cause the no-op instruction to be restored. 

The end result is a significant reduction in the overhead of disabled tracepoints. This feature only works on architectures which support it (x86 only, at the moment) and only with relatively recent versions of GCC; otherwise the preprocessor version is used.
