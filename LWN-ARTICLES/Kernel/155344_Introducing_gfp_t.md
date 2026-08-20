---
title: Introducing gfp_t
url: https://lwn.net/Articles/155344/
date: "October 11, 2005"
category: gfp t
author: 
---

Most kernel functions which deal with memory allocation take a set of "GFP flags" as an argument. These flags describe the allocation and how it should be satisfied; among other things, they control whether it is possible to sleep while waiting for memory, whether high memory can be used, and whether it is possible to call into the filesystem code. The flags are a simple integer value, and that leads to a potential problem: coding errors could result in functions being called with incorrect arguments. An occasional error has turned up where function arguments have gotten confused (usually through ordering mistakes). The resulting bugs can be strange and hard to track down. 

A while back, the `__nocast` attribute was added to catch these mistakes. This attribute simply says that automatic type coercion should not be applied; it is used by the `sparse` utility. A more complete solution is on the way, now, in the form of a new `gfp_t` type. The [patch](<https://lwn.net/Articles/155348/>) defining this type, and changing several kernel interfaces, was posted by Al Viro and merged just before 2.6.14-rc4 came out. There are several more patches in the series, but they have evidently been put on hold for now. 

The patches are surprisingly large and intrusive; it turns out that quite a few kernel functions accept GFP flags as arguments. For all that, the actual code generated does not change, and the code, as seen by `gcc`, changes very little. Once the patch set is complete, however, it will allow comprehensive type checking of GFP flag arguments, catching a whole class of potential bugs before they bite anybody.
