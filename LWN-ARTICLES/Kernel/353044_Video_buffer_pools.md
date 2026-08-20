---
title: Video buffer pools
url: https://lwn.net/Articles/353044/
date: "September 16, 2009"
category: Video4Linux2
author: 
---

The Video4Linux2 API has a well-developed interface for sharing video buffers between user space and the kernel. It is not without its problems, though. Simple video acquisition devices transfer large amounts of data (video frames) but cannot do scatter/gather I/O, forcing the allocation of large, physically-contiguous buffers. Queueing buffers for frame transfers can be a significant source of latency, especially when user-space buffers need to be locked into memory or when the architecture requires significant cache invalidation operations. It would also be nice to be able to pass buffers directly between video devices and related devices, such as hardware codecs, but the current API does not support that well. 

In response to these problems, Laurent Pinchart has [proposed](<https://lwn.net/Articles/353043/>) a new subsystem implementing a global video buffer pool. These buffers would be allocated early in the system's lifetime, working around the unreliability of large contiguous allocations. Cache invalidation operations could be done ahead of time, eliminating a significant source of capture-time latency. Passing buffers between devices would be explicitly supported. The proposal is in an early stage, and Laurent would like comments from interested developers.
