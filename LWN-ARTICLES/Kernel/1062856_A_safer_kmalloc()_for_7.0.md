---
title: A safer kmalloc() for 7.0
url: https://lwn.net/Articles/1062856/
date: "March 16, 2026"
category: "Memory management-Slab allocators; Releases-7.0"
author: "By Jonathan Corbet March 16, 2026"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 16, 2026

A pull request that touches over 8,000 files, changing over 20,000 lines of code in the process, is (fortunately) not something that happens every day. It did happen at the end of the 7.0 merge window, though, when Linus Torvalds [merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8934827db540>) an extensive set of changes by Kees Cook to the venerable `kmalloc()` API (and its users). As a result of that work, though, the kernel has a new set of type-safe memory-allocation functions, with a last-minute bonus change to make the API a little easier to use. 

#### Classic `kmalloc()`

`kmalloc()` is a general-purpose interface to the slab allocator; its purpose is to allocate small (generally sub-page) chunks of memory for use within the kernel. The first kernel release to contain `kmalloc()` was 0.98.4 from November 1992, though a similar function existed under the `malloc()` name since the 0.11 release at the end of 1991. The 0.98.4 version had a reasonably familiar prototype: 

```
void *kmalloc(unsigned int len, int priority);
```

The `len` parameter specifies how much memory is needed, while `priority` describes how the memory should be allocated; in 0.98.4 it could be one of `GFP_BUFFER`, `GFP_ATOMIC`, `GFP_USER`, or `GFP_KERNEL`. The return value, if all goes well, is a pointer to the newly allocated chunk of memory. 

In current kernels, instead, that prototype is: 

```
void *kmalloc(size_t size, gfp_t gfp);
```

The types of the arguments have shifted slightly, and the (now) `gfp` argument is an explicit bitmask, but otherwise it is essentially unchanged, more than 34 years later. 

The `kmalloc()` API clearly works, but it is also a 20th century C interface. Its return value is untyped, and there is nothing ensuring that the size of the allocated chunk of memory is correct. That leaves developers exposed to classic errors like this: 

```
struct foo *ptr;
    
        ptr = kmalloc(sizeof(ptr), GFP_KERNEL);  /* Don't do this */
```

The code is valid C, but it will successfully allocate a block of memory that is almost certainly too small (the size of a pointer rather than to the pointed-to type). Once developers start allocating arrays of objects the number of opportunities for mistakes that the compiler cannot detect grows even further; allocations of objects with flexible array members are more error-prone yet. Unsurprisingly, such mistakes have been the source of numerous kernel bugs during the history of the project. 

#### Safer memory allocation

Efforts have been made over the years to improve the safety of `kmalloc()`, with some success. Furthering that work, the 7.0 kernel release will include a relatively large change that is intended to make a lot of typical errors impossible. The new functions (more precisely, macros) were added by [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2932ba8d9c99>); they are: 

```
ptr = kmalloc_obj(*ptr, gfp);
        ptr = kmalloc_objs(*ptr, count, gfp);
        ptr = kzalloc_obj(*ptr, gfp);
        ptr = kzalloc_objs(*ptr, count, gfp);
```

The simplest form is functionally identical to a basic `kmalloc()` call, but there is no need to use `sizeof()`, and the type of the return value from the macro will be a pointer type matching the first parameter. So if a developer were to type: 

```
ptr = kmalloc_obj(ptr, GFP_KERNEL);  /* Argument should be "*ptr" */
```

The compiler will object with a complaint that the return value of `kmalloc_obj()` does not match the type of the pointer to which that value is being assigned. This version, in other words, has a level of type safety that `kmalloc()` has always lacked. 

Arrays of objects can be allocated with `kmalloc_objs()`, eliminating the need for the sort of arithmetic that has proved surprisingly difficult to get right over time. The `kzalloc_` versions will zero the allocated memory before returning it. 

Structures with [flexible array members](<https://lwn.net/Articles/908817/>) can be another source of allocation mistakes; developers will often get the calculation of the total structure size wrong. There is a new allocator (added in [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e4c8b46b924e>)) that is designed to eliminate those mistakes: 

```
ptr = kmalloc_flex(*ptr, flex_member, count, gfp);
```

Here, `*ptr` is a structure with a flexible array member, the name of which is `flex_member`. The number of elements that the flexible array should be sized for is given by `count`. The returned object will be sized correctly to hold the requested number of elements in its flexible array. As an added bonus, if the structure is defined using [`__counted_by()`](<https://lwn.net/Articles/936728/>) to indicate which member holds the size of the flexible array, that field will be automatically initialized during the allocation — at least, on compilers that fully support `__counted_by()`. 

With these new allocation functions in place, the stage was set to convert much of the existing kernel code base over to their use. That was the purpose of [the massive patch](<https://git.kernel.org/linus/69050f8d6d07>) mentioned in the introduction, as well as a number of [followup patches](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=189f164e573e>) cleaning up the harder cases. 

#### Implicit `GFP_KERNEL`

The large-scale patching was not quite done yet, though. As he pulled in Cook's changes, Torvalds [observed](<https://lwn.net/ml/all/CAHk-=wjOeEk1oFekLvEx081ZROBm3oHTeWzOg8W5AFyErsC57A@mail.gmail.com>) that almost all of the allocation calls use `GFP_KERNEL`; that result is unsurprising, since the more restrictive allocation options are only used when they are truly necessary. He wondered if, by way of some macro magic, the `gfp` argument could be made optional, with a default of `GFP_KERNEL` when it is not supplied. About nine hours later, he [reported](<https://lwn.net/ml/all/CAHk-=wh8Lc=OwB3QDssXWaEpntvYcEZH6GhmGZCewNctZo591Q@mail.gmail.com>) that he had implemented and applied that change. As he pointed out, there was no better time to thrash that much code: ""those lines are all being modified anyway, so any merge conflict pain is not going to be made worse if I tweak the end result a bit more"". 

That work resulted in [another massive commit](<https://git.kernel.org/linus/bf4afc53b77ae>) removing the unneeded `GFP_KERNEL` argument from `kmalloc_obj()` calls, and [a smaller one](<https://git.kernel.org/linus/323bbfcf1ef88>) for `kmalloc_flex()`. He declared victory before fixing every single call, but stated that he was happy with the results: ""The code really does look better"". After more than three decades, the kernel's core allocation mechanism for small objects finally looks a bit different, and is hopefully less susceptible to silly mistakes. So developers will have something to celebrate, even as they grumble about having to fix the countless merge conflicts this change has surely created.
