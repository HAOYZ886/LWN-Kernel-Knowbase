---
title: "A multi-order radix tree"
url: https://lwn.net/Articles/688130/
date: "May 24, 2016"
category: Radix tree
author: "May 24, 2016 This article was contributed by Ross Zwisler"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

May 24, 2016

This article was contributed by Ross Zwisler

Radix trees have been a part of Linux for quite some time; an [LWN article](<https://lwn.net/Articles/175432/>) from a decade ago explained the structure and functionality of them. The radix tree is a general-purpose data structure that is used by many different components within the kernel. It provides an efficient way to create a key-value store, where the key is an `unsigned long`, referred to as the `index`, and the value is a `void *`. The radix tree also stores a few bits of additional information for each entry in the form of tags. 

The most common use of the radix tree is to keep track of a collection of pages. In `struct address_space`, for example, there is a radix tree called `page_tree` that tracks the in-memory page-cache pages that are associated with a given inode. The key for `page_tree` is the page offset (`pgoff_t`) into the file. For normal files, `page_tree` will map that key to a `void *` value which is actually the `struct page *` for the page-cache page at that file offset. For page-cache pages, the radix-tree tags let us track entries that are dirty and which are marked for writeback. 

For inodes that take advantage of the [DAX](<https://lwn.net/Articles/610174/>) ("direct access") feature, there is no page cache sitting between the user processes and the storage. Hence, for DAX inodes there is no need to keep track of `struct page *` entries via the `page_tree`. Instead, for DAX inodes, this same `page_tree` is used to hold DAX exceptional entries that track the state of the persistent-memory pages used by DAX. On x86_64, these exceptional entries are 64-bit values that store several pieces of information, such as the page size (more on that below), the sector offset within the persistent-memory storage, and some flags. From these exceptional entries, DAX knows which dirty pages need to be flushed from the processor cache when an `fsync()` is received from user space. 

For radix-tree uses where there is a one-to-one mapping between keys and values, such as a `page_tree` that only tracks `PAGE_SIZE` page-cache entries and/or DAX entries, this all works perfectly. But what about cases where this one-to-one relationship breaks down? 

One example of this breakdown is huge pages. On x86_64, regular pages are 4KiB in size. Linux x86_64 also supports "huge pages" that are 2MiB in size, and the Linux DAX code has explicit support for these 2MiB pages. For the `page_tree` radix tree, this means that the one-to-one relationship between keys and values may not be sufficient. 

There is a desire for a 2MiB page to be tracked as a single entity. There would be a single pointer to the 2MiB worth of data and the tag state would be consistent, so that the kernel can reliably track whether the data is dirty. 

#### Existing solutions

As of kernel 4.5, DAX successfully tracks the value and state of 2MiB pages through the use of DAX exceptional radix-tree entries that reserve a few bits to record whether the DAX entry represents a 4KiB page (`RADIX_DAX_PTE`) or a 2MiB page (`RADIX_DAX_PMD`). 2MiB page entries (referred to as "Page Middle Directory" or PMD entries) are always inserted at a 2MiB boundary, so DAX is able to support huge-page entries with the following logic: 

```
pgoff_t pmd_index = DAX_PMD_INDEX(index);
    
        entry = radix_tree_lookup(page_tree, pmd_index);
    
        if (entry && RADIX_DAX_TYPE(entry) == RADIX_DAX_PMD)
    	    /* operate on the 2MiB 'entry' at 'pmd_index' */
        else
    	    entry = radix_tree_lookup(page_tree, index);
    	    /* operate on the 4KiB 'entry' at 'index' */
```

This has the obvious cost that for every radix-tree operation there has to be an extra lookup for the entry using `pmd_index` to first check whether the index is covered by a 2MiB page. This solution is correct, but is somewhat costly in the `RADIX_DAX_PTE` case where we have to do a `radix_tree_lookup()` at both `pmd_index` and `index`. Having special-case lookups at multiple offsets also does not make for the cleanest code. When 1GiB page support is added to the DAX code, this solution begins to look even worse, because there will have to be yet another special-case lookup. 

Another possible alternative that would not involve an extra lookup would be to represent the 2MiB entry via 512 redundant entries, each at a unique index. This would have the property that any lookup for the indices in the 2MiB range would return a copy of the data pointer, but it has the downside that the tag tracking is no longer consistent among the 512 entries. This would mean that some of the 512 entries could be tagged as clean and some of them could be tagged as dirty, even though they all had the same data pointer as their value. 

There would also be a need to be sure to replicate other operations, such as removal, among all 512 entries. This solution has the additional downside that representing a 2MiB page via 512 individual entries adds many extra nodes to the radix tree. 

#### Multi-order radix-tree techniques

Both of the solutions mentioned thus far for dealing with huge pages have left the radix-tree API unchanged. Ideally, there would be a solution where the one-to-one mapping between keys and values in the radix tree can be broken. That would allow inserting an entry that covers multiple 4KiB-sized indices and have operations on indices in that range, such as lookup, tag modification, and removal, all act on the same radix-tree entry. Recently there have been several patch series ([1](<https://lkml.org/lkml/2016/1/19/363>), [2](<https://lkml.org/lkml/2016/4/6/718>), and [3](<https://lkml.org/lkml/2016/4/14/522>)) posted to the Linux kernel mailing list that set out to accomplish this goal via a solution that is called "the multi-order radix tree". 

The basic idea is to add the ability to insert entries that span 2X 4KiB-sized page indices. X is referred to as the page's "order". Using this terminology, existing radix trees, in which every entry is associated with a single index, are composed entirely of order-0 entries. An order-2 entry would cover 22 = 4 adjacent indices. The 2MiB entries for the DAX huge-page example would be order-9 entries, and so on.

![\[sibling pointer\]](https://static.lwn.net/images/2016/mo_radix_tree_1.png) This new functionality is implemented in the multi-order radix tree using a pair of techniques: sibling entries and elevated entries. The smallest multi-order entry is an order-1 entry that covers two adjacent indices. This is implemented by inserting a special "sibling pointer" for the second index that points back to the actual radix-tree entry.

In this case, lookups, removals, and tag operations on both the base index and on the index for the sibling operate on the same actual entry in a way that is transparent to the user. For orders greater than 1, there can simply be multiple sibling entries that all point back to the actual radix-tree entry:  ![\[multi-order radix 2\]](https://static.lwn.net/images/2016/mo_radix_tree_2.png)

With a multi-level radix tree, there can be up to three different types of pointers. The first are internal pointers, which point from a parent radix-tree node to a child radix-tree node. The second are sibling pointers, which point from one entry in a given radix-tree node to another entry in that same node. The third are the `void *` data pointers that are stored as part of the key-value store. 

The lowest bit of the radix-tree entry, `RADIX_TREE_INTERNAL_NODE`, is used to distinguish between the `void *` data pointers and the two types of pointers internal to the radix-tree implementation. Sibling pointers and internal node-to-node pointers are distinguished by looking at the value of the pointer itself. If the pointer points within the same node's `slots` array, it is a sibling pointer. If not, it points to a child radix-tree node. 

If the fan-out of the radix tree happens to match the order of the multi-order entry, it can be represented using an elevated entry that simply lives as a child of one of the intermediate nodes in the tree: 

![\[multi-order radix 3\]](https://static.lwn.net/images/2016/mo_radix_tree_3.png)

Elevated multi-order entries can be the children of intermediate nodes at any level in the tree. Combining these two techniques allows us to have elevated multi-order entries with sibling pointers: 

![\[multi-order radix 4\]](https://static.lwn.net/images/2016/mo_radix_tree_4.png)

Having both sibling entries and elevated entries allows the radix tree to support multi-order entries of any order. 

#### Radix-tree API changes

To use this new functionality, the multi-order radix tree has a few small API changes where an `order` parameter was needed. 

```
int __radix_tree_create(struct radix_tree_root *root, unsigned long index,
    			    unsigned order, struct radix_tree_node **nodep,
    			    void ***slotp);
    
        int __radix_tree_insert(struct radix_tree_root *, unsigned long index,
    			    unsigned order, void *);
```

`__radix_tree_create()` is only used in one place in `mm/filemap.c`, and `__radix_tree_insert()` is a new API added by the multi-order patches. `radix_tree_insert(),` the old insertion API that is used by all existing code, is now defined to be: 

```
static inline int radix_tree_insert(struct radix_tree_root *root,
    			    unsigned long index, void *entry)
        {
    	    return __radix_tree_insert(root, index, 0, entry);
        }
```

The API for operations such as node lookup, deletion, tag manipulation, etc. remain unchanged. This has allowed the multi-order radix tree to be implemented with very little change to existing radix-tree users. 

#### Integration with DAX 2MiB support

I recently posted a [patch](<https://lkml.org/lkml/2016/4/14/865>) that integrates the DAX code with the new multi-order radix-tree code. As can be seen from that patch, the changes needed to move from the old method for supporting 2MiB pages to the new multi-order radix-tree support are quite small. 

We now insert an order-9 entry when we need to track the status of a 2MiB huge page. This is done as follows: 

```
error = __radix_tree_insert(page_tree, index,
                        RADIX_DAX_ORDER(pmd_entry),
                        RADIX_DAX_ENTRY(sector, pmd_entry));
```

`RADIX_DAX_ORDER()` gives us an order of 0 for 4KiB pages and an order of 9 for 2MiB pages. 

For the rest of the radix-tree operations, like lookup and tag manipulation, there is no longer a need to first check for a 2MiB PMD entry as a special case. It just operates on the radix tree using the index and the radix tree will do the right thing if that index happens to map to a multi-order entry. 

One last thing worth noting is that the multi-order radix tree API currently does not define a way for the user to query the order of a given entry. It is not immediately obvious whether this API is actually needed. The DAX code can infer the order of a given entry by looking at its type: `RADIX_DAX_PTE` or `RADIX_DAX_PMD`. When multi-order `struct page` entries start being inserted, their size can most likely be understood by looking at the page flags. However, if an API to query an entry's order proves useful, it could easily be added. 

#### In conclusion

The multi-order radix tree patches have been present in both Andrew Morton's -mm tree as well as Stephen Rothwell's linux-next tree for several weeks; they were pushed upstream during the 4.7 merge window. The integration between DAX PMD entries and the new multi-order radix-tree code will be merged in 4.8 or later, and will need to take into account the recent [DAX page-fault-locking](<https://lwn.net/Articles/687541/>) patch series from Jan Kara. The combination of the multi-order radix-tree patches and the locking changes will allow DAX to have locks on a per-page basis, regardless of the size of the page. 

DAX will most likely be the first user of the new multi-order capability of the radix tree, but these changes should be interesting to anyone who deals with multiple page sizes. The transparent-huge-page code could probably make use of this new functionality, and it is likely that other users will spring up over time.
