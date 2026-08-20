---
title: A new block request completion API
url: https://lwn.net/Articles/266914/
date: "January 29, 2008"
category: "Block layer-Block drivers; Device drivers-Block drivers"
author: "By Jonathan Corbet January 29, 2008"
---

By **Jonathan Corbet**  
January 29, 2008

The 2.6 block layer has traditionally provided a pair of functions by which a driver could indicate that an I/O request had been completed. A call to `end_that_request_first()` signaled the transfer of a certain amount of data and would return a value indicating whether the request as a whole was complete. Once all sectors in a request had been transferred, it was up to the driver to pass the request to `end_that_request_last()` for final cleanup. There was also a function called simply `end_request()` which might or might not end the entire request, depending on how much data had been transferred. This API has worked for a long time, but it has occasionally proved confusing for driver developers. It was also hard for drivers to communicate useful error information with this interface. So, as of 2.6.25, there will be a new way for drivers to indicate request completion. 

After a block driver has transferred one or more sectors (or failed in the attempt), it should now make a call to: 

```
int blk_end_request(struct request *rq, int error, int nr_bytes);
```

Where `rq` is the I/O request, `error` is zero or a negative error code, and `nr_bytes` is the number of bytes successfully transferred. If `blk_end_request()` returns zero, the request is fully processed and the driver can forget about it. Otherwise there are still sectors to be transferred and the driver should continue with the same request. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

`blk_end_request()` must acquire the queue lock to do its job. If the driver already holds that lock, it should call `__blk_end_request()` instead. 

Block drivers traditionally did a number of housekeeping tasks between calls to `end_that_request_first()` and `end_that_request_last()`. These include calling `add_disk_randomness()` to contribute to the entropy pool, returning any tags used with the request, and removing the request from the queue. All of that stuff is now done within `blk_end_request()`, so drivers can forget about it. The occasional driver had to carry out other tasks between the completion of the request and its removal from the queue. For drivers with this kind of special need, there is a separate function to call: 

```
int blk_end_request_callback(struct request *rq, 
                                     int error, 
    				 int nr_bytes,
    			         int (drv_callback)(struct request *));
```

In this version, `drv_callback()` will be called (without the queue lock held) between the completion of the request and its final cleanup. If the callback returns a non-zero value, that final cleanup will not be done. This function will always acquire the queue lock - there is no version for drivers which have already taken that lock. In general, though, the use of the callback functionality is likely to be a sign that the driver is being tricker than it really needs to be. 

This change was accompanied by a fair number of patches converting all in-tree drivers to the new interface. The old completion functions have been removed, so out-of-tree drivers will need updating before they will work with 2.6.25.
