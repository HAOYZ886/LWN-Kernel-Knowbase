---
title: "This week's version of the kevent interface"
url: https://lwn.net/Articles/208139/
date: "November 7, 2006"
category: Kevent
author: 
---

The proposed kevent interface was last [covered here](<http://lwn.net/Articles/196721/>) in August. This new API, which seeks to provide a single interface for applications to received events of interest, has been under development for the better part of a year now. It continues to evolve, so, in celebration of [the version 23 kevent patch](<http://lwn.net/Articles/208079/>), another look is called for. 

Parts of the interface remain relatively stable. So, the main multiplexer system call remains: 

```
int kevent_ctl(int fd, unsigned int cmd, unsigned int num,
                       struct ukevent *arg);
```

The functions performed by this call are reduced in number, however. It is no longer used to create the kevent file descriptor in the first place; instead, an open of `/dev/kevent` is called for. But `kevent_ctl()` is still the place to add events of interest, and to remove and modify them. 

The synchronous interface for waiting for events is also pretty much as it has been for a little while: 

```
int kevent_get_events(int fd, unsigned int min_nr, unsigned int max_nr,
                              __u64 timeout, struct ukevent *buf, 
    			  unsigned flags);
```

This system call will wait until at least `min_nr` events are ready for consumption, then copy up to `max_nr` completed events into `buf`. The call will return early, however, if `timeout` nanoseconds pass before `min_nr` events are signaled. The current [documentation](<http://linux-net.osdl.org/index.php/Kevent>) for kevents says that an indefinite wait can be had by passing `-1` for `timeout` \- slightly strange, given that `timeout` is an unsigned quantity. It would not be surprising to see some sort of `KEVENT_WAIT_FOREVER` value defined for that purpose instead. 

The biggest changes can be found in the kevent ring buffer code which, last time we looked, was rather awkward to use. The previous implementation also placed the ring buffer in nailed-down kernel memory, potentially opening the system up to denial of service problems. So, in the new implementation, the ring buffer is kept entirely in user space. The application simply allocates an array of the desired size with the following type: 

```
struct kevent_ring
        {
    	unsigned int		ring_kidx;
    	struct ukevent		event[0];
        };
```

The actual number of events to be stored in the ring is determined by the application. The kevent subsystem must be told about this ring with: 

```
int kevent_ring_init(int fd, struct kevent_ring *ring, 
                             unsigned int num);
```

where `num` is the number of `ukevent` structures in the ring. This call will remember the ring's address and size, and set `ring_kidx` \- the index of the entry where the kernel will store the next completed event - to zero. 

There are a few things to be aware of when working with the kevent ring. One is that there is no place in this data structure to track which event the application should consume next; the application must store that index elsewhere. There also appears to be no way to disconnect or resize the ring buffer without simply closing the event file descriptor and starting over; an attempt to replace one ring with another will fail. Finally, the application must tell the kernel to put events into the ring with: 

```
int kevent_wait(int fd, unsigned int num, __u64 timeout);
```

This system call will wait until at least one event is available, then copy up to `num` events into the ring buffer. Once the events are copied, the kernel considers them to be consumed and will forget about them (or requeue them if the event so requests). The application can work through the events at leisure - stopping before hitting the current `ring_kidx` value - with no further system calls required. 

The current API seems to have made most of the people who are paying attention happy - though it has been a little while since Ulrich Drepper, an important player, has chimed in. In the past, he has been unhappy about the timeout parameter (preferring that the interface use an absolute `timespec` value rather than a relative value). Ulrich has also suggested that the blocking system calls could use a version which specifies an event mask, much like the recently added `ppoll()` and `pselect()` system calls. He points out that, while it is possible to receive signals as kevents, some applications will certainly still use traditional signals, with their traditional atomicity problems. 

So there may be a few remaining issues to take care of before the kevent API is merged into the mainline kernel - and consequently set in stone. But there is apparent progress in that direction, and the number of developers showing interest in this API appears to be on the increase. It may not be too many more kernel cycles before Linux has a unified event interface of its very own.
