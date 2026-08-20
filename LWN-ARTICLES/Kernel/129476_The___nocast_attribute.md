---
title: The __nocast attribute
url: https://lwn.net/Articles/129476/
date: "March 29, 2005"
category:   nocast; Debugging
author: 
---

Attentive readers of patches being merged for 2.6.12-rc2 will have noticed the use of a new attribute: `__nocast`. For example, the prototype of `kmalloc()` has changed to: 

```
void *kmalloc(size_t size, unsigned int __nocast flags);
```

For normal compilation, this attribute expands to an empty string; it has no effect. When the [`sparse`](<https://lwn.net/Articles/87538/>) tool is being used, however, the `__nocast` attribute disables many of the implicit type conversions performed by the compiler. In the `kmalloc()` case, `sparse` will complain whenever a signed integer value is passed as the `flags` argument. Since the GFP flags passed to `kmalloc()` are explicitly defined as unsigned values, they will not cause a warning to be issued. Any normal integer variable or constant, however, will be flagged. Similarly, the use of an integer value where an enumerated type is expected will be caught. Thus, this little tweak should help with the automated detection of another class of errors that the compiler will not find.
