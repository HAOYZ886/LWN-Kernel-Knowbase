---
title: The XArray data structure
url: https://lwn.net/Articles/745073/
date: "January 24, 2018"
category: Radix tree; XArray
author: "By Jonathan Corbet January 24, 2018 linux.conf.au"
---

By **Jonathan Corbet**  
January 24, 2018

* * *

[linux.conf.au](<https://lwn.net/Archives/ConferenceByYear/#2018-linux.conf.au>)

Sometimes, a data structure proves to be inadequate for its intended task. Other times, though, the problem may be somewhere else — in the API used to access it, for example. Matthew Wilcox's presentation during the 2018 linux.conf.au Kernel miniconf made the case that, for the kernel's venerable [radix tree](<https://lwn.net/Articles/175432/>) data structure, the latter situation holds. His response is a new approach to an old data structure that he is calling the "XArray". 

The kernel's radix tree is, he said, a great data structure, but it has far fewer users than one might expect. Instead, various kernel subsystems have implemented their own data structures to solve the same problems. He tried to fix that by converting some of those subsystems and found that the task was quite a bit harder than it should be. The problem, he concluded, is that the API for radix trees is bad; it doesn't fit the actual use cases in the kernel. 

Part of the issue is that the "tree" terminology is confusing in this case. A radix tree isn't really like the classic trees that one finds in data-structure texts. Addition of an item to a tree has been called "insertion" for decades, for example, but an "insert" doesn't really [![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2018/lca/MatthewWilcox-sm.jpg)](<https://lwn.net/Articles/745075/>) describe what happens with a radix tree, especially if an item with the given key is already present there. Radix trees also support concepts like "exception entries" that users find scary just because of the naming that was used. 

So Wilcox decided to fix the interface. He has kept the existing radix-tree data structure unchanged; there are, he said, few problems with it. But the metaphor describing its operation has been changed from a tree to an array. It behaves much like an automatically resizing array; fundamentally, it is an array of pointer values indexed by an unsigned long. This view better describes how the structure is actually used. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The radix tree requires users to do their own locking; the XArray, instead, handles locking itself by default, simplifying the task of using it. The "preload" mechanism, which allows users to pre-allocate memory before acquiring locks, has been removed; it added significant complexity to the interface for almost no real value. 

The actual XArray API has been split into two pieces, the normal API and the advanced API. The latter provides much more control to the caller; it can be used to explicitly manage locking, for example. This API will be used at call sites with special needs; the page cache is one example where it is needed. The normal API is entirely implemented on top of the advanced API, so it serves as a demonstration of how the advanced API can be used. 

The page cache has been converted to use the XArray, he said, and there are no bugs remaining that he knows of. His plan is to "plead" for inclusion during the 4.16 merge window. 

#### A quick look at the XArray API

The current version of the XArray patch set, as of this writing, is [version 6](<https://lwn.net/Articles/744647/>), posted on January 17\. It is a 99-patch series and, thus, not for the faint of heart, but an introduction to its operation can be found in [the documentation patch](<https://lwn.net/Articles/745074/>) in the series. One starts by defining an array with: 

```
#include <linux/xarray.h>
    
        DEFINE_XARRAY(array_name);
        /* or */
        struct xarray array;
        xa_init(&array);
```

Storing a value into an XArray is done with: 

```
void *xa_store(struct xarray *xa, unsigned long index, void *entry,
                       gfp_t gfp);
```

This function will store the given `entry` at the requested `index`; if memory must be allocated, the given `gfp` flags will be used. The return value on success is the previous value (if any) that was stored at `index`. An entry can be removed from the array by storing `NULL` there, or by calling: 

```
void *xa_erase(struct xarray *xa, unsigned long index);
```

Other variants include `xa_insert()` to store without overwriting an existing entry, and `xa_cmpxchg()`: 

```
void *xa_cmpxchg(struct xarray *xa, unsigned long index, void *old,
                         void *entry, gfp_t gfp);
```

In this case, `entry` will be stored at `index`, but only if the current value stored there matches `old`. Either way, the current value stored at `index` is returned. 

Fetching a value from an XArray is done with `xa_load()`: 

```
void *xa_load(struct xarray *xa, unsigned long index);
```

The return value is the value stored at `index`. In an XArray, an empty entry is the same as an entry that has had `NULL` stored into it, so `xa_load()` will not behave specially for empty entries. 

Up to three single-bit tags can be set on any non-null XArray entry; they are managed with: 

```
void xa_set_tag(struct xarray *xa, unsigned long index, xa_tag_t tag);
        void xa_clear_tag(struct xarray *xa, unsigned long index, xa_tag_t tag);
        bool xa_get_tag(struct xarray *xa, unsigned long index, xa_tag_t tag);
```

The `tag` values here are one of `XA_TAG_0`, `XA_TAG_1`, and `XA_TAG_2`. A call to `xa_set_tag()` will set the given `tag` on the entry at `index`, while `xa_clear_tag()` will remove that tag. `xa_get_tag()` returns true if the given `tag` is set on the entry at `index`. 

As a general rule, an XArray is sparsely populated; that means that looping through all of the possible entries looking for the non-null ones would be rather inefficient. Instead, this helper macro should be used: 

```
xa_for_each(xa, entry, index, max, filter) {
            /* Process "entry" */
        }
```

Before entering this loop, `index` should be set to the beginning of the range to be iterated over, while `max` indicates the largest index that should be returned. The `filter` value can specify tag bits which will be used to filter out uninteresting entries. During loop execution, `index` will be set to match the current entry. It is possible to change the iteration by changing `index` within the loop; it is also allowed to make changes to the array itself. 

There are many other functions in the normal API that provide other ways to access an XArray; there is also the entire advanced API for the special cases. The API as a whole is reasonably large and complex, but it would appear to be rather easier to work with than the radix-tree API. The current patch set converts a number of radix-tree users to XArrays, but some still remain. If all goes according to Wilcox's plan, though, those will be converted in the near future and the radix-tree API will head toward removal. 

[Your editor would like to thank the Linux Foundation, LWN's travel sponsor, and linux.conf.au for assisting with his travel to the event.]
