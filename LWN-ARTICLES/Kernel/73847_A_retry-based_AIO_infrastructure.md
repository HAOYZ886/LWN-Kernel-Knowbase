---
title: "A retry-based AIO infrastructure"
url: https://lwn.net/Articles/73847/
date: "March 2, 2004"
category: Asynchronous IO
author: 
---

The asynchronous I/O infrastructure was added in 2.5 as a way to allow processes to initiate I/O operations without having to wait for their completion. The underlying mechanism is documented in [this Driver Porting Series article](<http://lwn.net/Articles/24366/>). The actual implementation of asynchronous I/O in the kernel has been somewhat spotty, however. It works for some devices (which have specifically implemented that support) and for direct file I/O. Other sorts of potentially interesting uses, such as with regular buffered file I/O, have remained unimplemented. 

Part of the problem is that buffered file I/O integrates deeply with the page cache and virtual memory subsystem. It is not all that easy to graft asynchronous I/O operations into those complex bodies of code. So the kernel developers have, for the most part, simply punted on cases like that. 

Suparna Bhattacharya, however, has not given up so easily. For over a year, now, she has been working on a set of patches which bring the asynchronous mode to the buffered I/O realm. [A new set of patches](<http://lwn.net/Articles/72678/>) has recently been posted which trims down the buffered AIO changes to the bare minimum. So this seems like a good time to take a look at what is involved in making asynchronous buffered I/O work. 

The architecture implemented by these patches is based on retries. When an asynchronous file operation is requested, the code gets things started and goes as far as it can until something would block; at that point it makes a note and returns to the caller. Later, when the roadblock has been taken care of, the operation is retried until the next blocking point is hit. Eventually, all the work gets done and user space can be notified that the requested operation is complete. The initial work is done in the context of the process which first requested the operation; the retries are handled out of a workqueue. 

For things to work in this mode, kernel code in the buffered I/O path must be taught not to block when it is working on an asynchronous request. The first step in this direction is the concept of an asynchronous wait queue entry. Wait queue entries are generally used, surprisingly, for waiting; they include a pointer to the process which is to be awakened when the wait is complete. With the AIO retry patch, a wait queue entry which has a `NULL` process pointer is taken to mean that actually waiting is not desired. When this type of wait queue entry is encountered, functions like `prepare_to_wait()` will not put the process into a sleeping state (though it does add the wait queue entry to the associated wait queue), and some functions will return the new error code `-EIOCBRETRY` rather than actually sleeping. 

The next step is to add a new `io_wait` entry to the task structure. When AIO retries are being performed, that entry is pointed to an asynchronous wait queue entry associated with the specific AIO request. This task structure field is, for all practical purposes, being used in a hackish manner to pass the wait queue entry into functions deep inside the virtual memory subsystem. It might have been clearer to pass it explicitly as a parameter, but that would require changing large numbers of internal interfaces to support a rarely-used functionality. The `io_wait` solution is arguably less clean, but it also makes for a far less invasive patch. It does mean, however, that work can only proceed on a single AIO request at a time. 

Finally, a few low-level functions have been patched to note the existence of a special wait queue entry in the `io_wait` field and to use it instead of the local entry that would normally have been used. In particular, page cache functions like `wait_on_page_locked()` and `wait_on_page_writeback()` have been modified in this way. These functions are normally used to wait until file I/O has been completed on a page; they are the point where buffered I/O often blocks. When AIO is being performed, instead, they will return the `-EIOCBRETRY` error code immediately. 

The AIO code also takes advantage of the fact that wait queue entries, in 2.6, contain a pointer to the function to be called to wake up the waiting process. With an asynchronous request, there may be no such process; instead, the kernel needs to attempt the next retry. So the AIO code sets up its own wakeup function which does not actually wake any processes, but which does restart the relevant I/O request. 

Once that structure is in place, all that's left is a bit of housekeeping code to keep track of the status of the request between retries. This work is done entirely within the AIO layer; as each piece of the request is satisfied, the request itself as seen by the filesystem layer is modified to take that into account. When the operation is retried to transfer the next chunk of data, it looks like a new request with the already-done portion removed. 

Add in a few other hacks (telling the readahead code about the entire AIO request, for example, and an AIO implementation for pipes) and the patch set is complete. It does not attempt to fix every spot which might block (that would be a large task), but it should take care of the most important ones.
