---
title: snprintf() confusion
url: https://lwn.net/Articles/69419/
date: "February 3, 2004"
category: scnprintf
author: 
---

Any C coder worth his or her salt knows that encoding text into a string with `sprintf()` invites buffer overflows, and is thus dangerous. The proper way of doing things is with `snprintf()`, which takes the length of the destination string as a parameter, and will not overrun it. Callers to `snprintf()` generally assume that the return value is the length of what was actually encoded into the destination array. That turns out, however, to not be the case. As per the C99 standard, `snprintf()` returns the length the resulting string _would_ be, assuming it all fit into the destination array. As a result of this misunderstanding, the kernel is full of `snprintf()` calls which use the return value incorrectly. 

This mistake is rarely a problem; `snprintf()` almost never has to truncate its output, so the return value is what the programmer is expecting. Every miscoded use is an invitation for trouble, however, and really should be fixed. To that end, the [2.6.2-rc3-mm1](<https://lwn.net/Articles/69348/>) tree contains a patch by Juergen Quade which adds a couple of new functions: 

```
int scnprintf(char *buf, size_t size, const char *format, ...);
        int vscnprintf(char *buf, size_t size, const char *format, va_list args);
```

The new functions work the way many programmers expected the old ones to: they return the length of the string actually created in `buf`. The plan is to migrate the kernel over to the new functions; the patch fixes well over 200 `snprintf()` and `vsnprint()` calls. Unless the old functions are eventually removed, however, they are likely to be a source of programming errors well into the future.
