---
title: On not getting burned by kmap_atomic()
url: https://lwn.net/Articles/111226/
date: "November 16, 2004"
category: Debugging; kmap atomic
author: 
---

"High memory," on a Linux system is, by definition, memory which is not normally mapped into the kernel's virtual address space. It is a mechanism which enables 32-bit architectures to make use of more physical memory than would otherwise be possible. When the kernel needs to directly manipulate the contents of a high-memory page, it must explicitly create a virtual address for it. The traditional functions for creating and removing those addresses are: 

```
void *kmap(struct page *page);
        void kunmap(struct page *page);
```

These functions work as intended, but they can be expensive to use. The virtual address space they use is limited, and shared across all processors. As a result, each `kmap()` and `kunmap()` invocation requires a global TLB flush. Often, however, high memory does not need to be mapped for long periods of time, and does not need to be shared across processors. To improve performance in such situations, the notion of an "atomic kmap" was added: 

```
void *kmap_atomic(struct page *page, enum km_type type);
        void kunmap_atomic(void *address, enum km_type type);
```

Atomic kmaps use a very small set of predefined virtual "slots," which are not shared across processors. The `type` argument specifies which slot is to be used, with the callers taking responsibility for not stepping on each others' toes. Slots are dedicated to specific purposes - two for code called in user context, two for interrupt handlers, two for page table management, etc. In practice, it all works out; conflicts over atomic kmap slots don't happen. 

Another problem _has_ come up, however, and that has led to a small change in the prototypes of the atomic kmap functions in the -mm kernel. The regular kmap functions have a symmetrical interface in that both take a `struct page *` argument. `kunmap_atomic()`, instead, takes a `void *` argument - the kernel virtual address to be unmapped. It is a common mistake, however, to pass in the associated `struct page` pointer instead. Since the argument type is `void *`, the compiler does not complain, and the discovery of the problem does not come until (possibly much) later. 

The solution is straightforward: redefine the function as follows: 

```
char *kmap_atomic(struct page *page, enum km_type type);
        void kunmap_atomic(char *address, enum km_type type);
```

With this change, the compiler will issue a warning whenever somebody tries to pass a `struct page` pointer to `kunmap_atomic()`. 

The patch has generated a surprising number of follow-on fixes, mostly to suppress warnings caused by the change. Many `kunmap_atomic()` calls now explicitly cast the address argument to the `char *` type. In the end, though, the result should be one more potential mistake which can be caught before it burns somebody - as long as programmers don't "fix" warnings by casting `struct page` pointers.
