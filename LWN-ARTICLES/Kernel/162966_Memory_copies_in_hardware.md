---
title: Memory copies in hardware
url: https://lwn.net/Articles/162966/
date: "December 7, 2005"
category: Direct memory access; IO AT
author: 
---

Upcoming versions of Intel processors will include a feature called an "asynchronous DMA engine." Essentially, it is a hardware peripheral which can be used to quickly copy data from one memory location to another. The "I/OAT" ("I/O acceleration technology") is expected to improve performance by offloading copy operations, enabling quick in-memory scatter/gather operations, and keeping copy operations from pushing useful data out of the processor's cache. 

Hardware with an I/OAT is not yet available, but [a patch for I/OAT support](<http://lwn.net/Articles/161671/>) has recently been posted. It lacks the hardware-level interface, but does demonstrate the API that the folks at Intel have come up with for this sort of device. 

Code which wishes to make use of the I/OAT must first register itself as a "DMA client." The registration interface looks like: 

```
#include <linux/dmaengine.h>
        typedef void (*dma_event_callback)(struct dma_client *client, 
                                           struct dma_chan *chan, 
    				       enum dma_event_t event); 
    
        struct dma_client *dma_async_client_register(dma_event_callback event_callback);
        void dma_async_client_unregister(struct dma_client *client);
```

The client must provide a callback function which will be invoked when DMA channels come and go. If all goes well, registration results in a `dma_client` structure which can be used with subsequent operations. 

Before anything can be done, the client must request one or more "channels." Every channel on the I/OAT can be used for one copy operation at a time; all channels can be operating simultaneously. The function to request channels is: 

```
dma_async_client_chan_request(struct dma_client *client, 
                                      unsigned int number);
```

The client's callback function will be called once for each allocated channel. The number of channels actually allocated may be less than what has been requested. There is no real guidance on the optimal number of channels to ask for; the example patch for the networking subsystem requests one channel for each processor on the system. The number of channels can be changed later on if need be. 

There are three functions for actually starting a copy operation: 

```
dma_cookie_t dma_async_memcpy_buf_to_buf(struct dma_chan *chan,
                                                 void *dest, void *src,
                                                 size_t len);
        dma_cookie_t dma_async_memcpy_buf_to_pg(struct dma_chan *chan,
                                                struct page *page,
                                                unsigned int offset,
                                                void *kdata, size_t len);
        dma_cookie_t dma_async_memcpy_pg_to_pg(struct dma_chan *chan,
                                               struct page *dest_pg,
                                               unsigned int dest_off,
                                               struct page *src_pg,
                                               unsigned int src_off,
                                               size_t len);
```

All three functions do the same thing: they request an asynchronous copy operation from one memory location to another. The only difference is whether kernel addresses or `page` structures are used to specify the locations. For some reason, it appears to be necessary to issue a call to: 

```
void dma_async_memcpy_issue_pending(struct dma_chan *chan);
```

before the operation will actually happen. 

Since copy operations are asynchronous, they may not have completed when the request functions return, so the caller should not mess with the affected buffers in the mean time. There are two functions for querying and waiting for completion: 

```
dma_async_memcpy_complete(struct dma_chan *chan, dma_cookie_t cookie,
                                  dma_cookie_t *last, dma_cookie_t *used);
        dma_async_wait_for_completion(struct dma_chan *chan, 
                                      dma_cookie_t cookie);
```

`dma_async_memory_complete()` will return one of `DMA_SUCCESS`, `DMA_IN_PROGRESS`, or `DMA_ERROR`, depending on the status of the copy operation indicated by `cookie` (the `last` and `used` arguments can be passed as `NULL`; their purpose is not entirely clear to your slow editor). A call to `dma_async_wait_for_completion()` will wait until the given operation finishes. In the current implementation, that wait is accomplished via a busy loop calling `schedule()`. There is no function for canceling an outstanding operation. 

The initial reaction to the patch was cautiously positive. There is some concern that invoking an external device to perform copies may be sufficiently expensive that it will only be worthwhile for very large operations. There were also some requests to extend the interface to include a transformation to be performed on the data as it is copied. The current hardware does not look like it will support anything beyond a direct copy (though, since the hardware is not yet available, it is hard to be sure), but it would be nice to be able to make use of any such capabilities as they arrive. Transformations could be simple (simply zeroing a buffer, say), or complex (cryptographic operations). But they will only be available if the interface supports them. 

The hardware is due in "early 2006," so more information will become available then. Until that time, there probably will not be any serious discussion of merging the I/OAT interface.
