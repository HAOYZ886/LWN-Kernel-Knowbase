---
title: Protecting private structure members
url: https://lwn.net/Articles/670237/
date: "January 6, 2016"
category: Development tools; sparse
author: "By Jonathan Corbet January 6, 2016"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 6, 2016

Most languages designed in the last few decades offer a way to restrict access to portions of a data structure, limiting their use to the code that directly implements the object that structure is meant to represent. The C language, initially designed in 1972, lacks any such feature. Most of the time, C (along with the projects using it) muddles along without this kind of protective feature. But that doesn't mean there would not be a use for it. 

If one browses through the kernel code, it's easy to find comments warning of dire results should outside code touch certain structure fields. The [definition of `struct irq_desc`](<http://lxr.free-electrons.com/source/include/linux/irqdesc.h#L46>) takes things a bit further, with a field defined as: 

```
unsigned int core_internal_state__do_not_mess_with_it;
```

Techniques like this work most of the time, but it would still be nice if the computer could catch accesses to structure members by code that should have no business touching them. 

Adding that ability is the goal of [this patch set](<https://lwn.net/Articles/670238/>) from Boqun Feng. It takes advantage of the fact that the venerable [`sparse` utility](<https://lwn.net/Articles/87538/>) allows variables to be marked as "not to be referenced." That feature is used primarily to detect kernel code that directly dereferences user-space pointers, but it can also be used to catch code that is messing around with structure members that it has not been invited to touch. Not all developers routinely run `sparse`, but enough do that new warnings would eventually be noticed. 

The patch set adds a new `__private` marker that can be used to mark private structure members. So the above declaration could become: 

```
unsigned int __private core_internal_state__do_not_mess_with_it;
```

As far as the normal C compiler is concerned, `__private` maps to the empty string and changes nothing. But when `sparse` is run on the code, it notes that the annotated member is not meant to be accessed and will warn when anybody tries. 

Of course, _some_ code must be able to access that field, or there is little point in having it there. Doing so without generating a `sparse` warning requires first removing the `__private` annotation; that is done by using the `ACCESS_PRIVATE()` macro. So code that now looks like: 

```
foo = s->private_field;
```

would have to become: 

```
foo = ACCESS_PRIVATE(s, private_field);
```

This aspect of the patch could prove to be the sticking point: some code may require a large number of `ACCESS_PRIVATE()` casts. Whether they are added to the code directly or hidden behind helper functions, they could lead to a fair amount of code churn if this feature is to be widely used. Given that the honor system works most of the time and that problems from inappropriate accesses to private data are relatively rare, the development community may decide that the current system works well enough.
