---
title: Fear of the void
url: https://lwn.net/Articles/88767/
date: "June 9, 2004"
category: ALSA; Coding style
author: 
---

When a kernel development project lives outside of the mainline kernel tree for a long time, it often picks up its own coding conventions which do not always match well with the kernel as a whole. One such project is the ALSA sound system, which was developed independently for years until it reached a state where it seemed ready to replace the old OSS drivers; it was merged in 2.5. Now some of the kernel developers are taking a look at the ALSA code and finding things which would, most likely, not have survived for long had ALSA been an in-tree development from the beginning. 

One of those is the ALSA convention for dealing with driver-private data. Many structures and callbacks in the kernel support the passing of private data; this is accomplished by way of a `void *` pointer. Creators and users of private data passed in this way are responsible for knowing what kind of structure is being dealt with and performing the appropriate casts. In general, this mechanism works well; there have been very few bugs resulting from confusion over the type of a private data pointer. 

Even so, the ALSA developers took some extra steps to ensure that errors do not creep in when private data is passed around; their conventions are [documented](<http://www.alsa-project.org/~iwai/writing-an-alsa-driver/x254.htm#CARD-MANAGEMENT-CHIP-WHAT-ADVANTAGE>) in the ALSA driver writing manual. In brief, it works as follows. The first step is to define a structure to be used as private data, create a type for it, and assign a magic number; the code tends to look like this: 

```
typedef struct { /* ... */ } funky_struct_t;
        #define funky_struct_t_magic 0x19980122
```

The value of the magic number is arbitrary (but should be unique); the name must match the defined type of the structure, however. 

When one of these structures is to be allocated, one of the following macros must be used: 

```
void *snd_magic_kmalloc(type, unsigned int extra-data, 
                                unsigned int flags);
        void *snd_magic_kcalloc(type, unsigned int extra-data, 
                                unsigned int flags);
```

The second version simply zeroes out the memory before returning it. Both versions allocate some extra space to store the magic number, thus identifying the allocated memory as holding a structure of the given type. 

When one of these structures is to be obtained from a `void *` private data pointer, the cast must be done in a special way: 

```
funky_struct_t *mydata;
        mydata = snd_magic_cast(funky_struct_t, void_pointer, return -ESCREWEDUP);
```

This macro will ensure that the types match; the final parameter is a line of C code to be executed should a mismatch occur. There is also, of course, a `snd_magic_kfree()` for freeing these structures. 

Attention was recently drawn to these conventions as part of an unrelated critique of the ALSA code. The kernel hackers, as a whole, do not like the "`snd_magic_`" macros; they feel that the rest of the kernel has gotten by just fine without that sort of infrastructure. It has also been noted that this kind of checking, if it is determined to be useful, should really be part of the central memory allocator rather than being specific to one subsystem. 

In response to the discussion, one energetic hacker has already sent out [a set of patches](<https://lwn.net/Articles/88778/>) removing most of the ALSA "magic" framework. ALSA maintainer Jaroslav Kysela has [requested](<https://lwn.net/Articles/88779/>) that they not be applied at this time, however; the ALSA team would like to figure out how best to clean up that code on its own. This effort may involve simply removing it, or [replacing it](<https://lwn.net/Articles/88788/>) with a less "magic" mechanism. One way or another, the ALSA code in the future will likely look more like the rest of the kernel than it does now.
