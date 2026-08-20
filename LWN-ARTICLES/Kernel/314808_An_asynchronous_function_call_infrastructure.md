---
title: An asynchronous function call infrastructure
url: https://lwn.net/Articles/314808/
date: "January 13, 2009"
category: Asynchronous function calls; Bootstrap process; Fast booting
author: "By Jonathan Corbet January 13, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 13, 2009

Arjan van de Ven's [fast boot project](<http://lwn.net/Articles/299483/>) will be familiar to most LWN readers by now. Most of Arjan's work has not yet found its way into the mainline, though, so most of us still have to wait for our systems to boot the slow way. That said, the 2.6.29 kernel will contain one piece of the fast boot work, in the form of the asynchronous function call infrastructure. Users will need to know where to find it, though, before making use of it. 

There are many aspects to the job of making a system boot quickly. Some of the lowest-hanging fruit can be found in the area of device probing. Figuring out what hardware exists on the system tends to be a slow task at best; if it involves physical actions (such as spinning up a disk) it gets even worse. Kernel developers have long understood that they could gain a lot of time if this device probing could, at least, be done in a parallel manner: while the kernel is waiting for one device to respond, it can be talking to another. Attempts at parallelizing this work over the years have foundered, though. Problems with device ordering, concurrent access, and more have adversely affected system stability, with the inevitable result that the parallel code is taken back out. So early system initialization remains almost entirely sequential. 

Arjan hopes to succeed where others have failed by (1) taking a carefully-controlled approach to parallelization which doesn't try to parallelize everything at once, and (2) an API which attempts to hide the effects of parallelization (other than improved speed) from the rest of the system. For (1), Arjan has limited himself to making parts of the SCSI and libata subsystems asynchronous, without addressing much of the rest of the system. The API work ensures that device registration happens in the same order is it would in a strictly sequential system. That eliminates the irritating problems which result when one's hardware changes names from one boot to the next. 

The API is relatively simple. The code needs to include `<async.h>` and create an asynchronous worker function matching this prototype: 

```
typedef void (async_func_ptr) (void *data, async_cookie_t cookie);
```

Here, `data` will be a typical private data pointer, and `cookie` is an opaque synchronization value passed in by the kernel. An asynchronous function call is made with a call to: 

```
async_cookie_t async_schedule(async_func_ptr *ptr, void *data);
```

The call to the function identified by `ptr` will happen sometime during or after the call to `async_schedule()`; in some circumstances, it may happen synchronously. The return value is a cookie identifying this particular asynchronous call. 

Code which calls asynchronous functions will eventually want to ensure that those functions have completed. The way to do that is through a call to: 

```
void async_synchronize_cookie(async_cookie_t cookie);
```

After this call completes, all asynchronous functions called prior to the one identified by `cookie` are guaranteed to have completed. Code which makes globally-visible changes (registering devices, for example) should synchronize in this manner first. In so doing, they ensure that any global changes which would have happened first in a strictly-sequential system will happen first in the asynchronous mode as well. 

Code wanting to wait for all asynchronous functions to complete can call: 

```
void async_synchronize_full(void);
```

This function returns when there are no asynchronous function calls in the system. Of course, another one could always be submitted immediately thereafter. 

Internally, the implementation of asynchronous functions is reasonably simple. There a pair of linked lists - `async_pending` and `async_running` \- containing pending and running function calls, respectively. A call to `async_schedule()` puts the call onto the pending list and, possibly, starts a kernel thread to get the job done. In general, there will be as many threads as there are outstanding asynchronous function calls, within a hard-coded maximum (currently 256). If a thread completes a function call and finds the pending list to be empty, it will exit. 

There is a special-purpose variation of this API: 

```
async_cookie_t async_schedule_special(async_func_ptr *ptr, void *data, 
                                              struct list_head *running);
        void async_synchronize_cookie_special(async_cookie_t cookie, 
        	 				  struct list_head *running);
        void async_synchronize_full_special(struct list_head *list);
```

These functions allow the caller to provide a separate list to be used in place of the `async_running` list. That, in turn, allows them to be synchronized independently of any other asynchronous functions running in the system. In 2.6.29-rc1, there is one prospective user of this API, which is, in fact, not part of the bootstrap process: the inode deletion code in the virtual filesystem layer. Making deletion asynchronous speeds up the process of deleting large numbers of files. It's worth noting that, in 2.6.29, this API also does not work quite as advertised - a shortcoming which, presumably, will be fixed soon. 

In fact, asynchronous function calls in general don't work as well as one might have liked at the moment. This code was merged for 2.6.29-rc1, but users immediately started reporting problems. One of those (which your editor stumbled across) is that the process of enumerating SATA disks can be "synchronized" while the partition enumerating process is still running, leading to systems which fail to boot. As a result of this problem and some other concerns, Arjan [asked](<https://lwn.net/Articles/314824/>) Linus to disable most of the code so that it could be stabilized for 2.6.30. In the end, the code remains in place, but it is not activated in the absence of the new `fastboot` kernel parameter. So adventurous developers can give asynchronous function calls a try; the rest of us can wait for this feature to cook just a little longer.
