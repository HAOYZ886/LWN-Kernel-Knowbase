---
title: Two API changes in 2.6
url: https://lwn.net/Articles/65163/
date: "January 6, 2004"
category: nopage
author: 
---

The kernel developers usually try to keep the internal kernel programming interface unchanged over the course of a stable kernel series. There are never any guarantees, however, and things can change at any time. Experience has shown, in particular, that internal APIs can take a little while to stabilize after a new stable series begins. The 2.6 kernel looks like it will follow this pattern; a couple of small changes have already found their way into the code base. 

The first is a simple addition: 

```
int can_request_irq(unsigned int irq, unsigned long flags);
```

This function will return a non-zero value if an attempt to request the given interrupt number (possibly shared, as directed by `flags`) would succeed. It is intended to be used in situations where multiple interrupt numbers could be used and the code would like to find an idle one. There are, of course, no guarantees; a kernel routine could get a positive result from `can_request_irq()`, but find that somebody else had slipped in and allocated the request number immediately thereafter. As of this writing, `can_request_irq()` is not exported to modules and is not supported by all architectures. 

The other change has the potential to create minor trouble for some external modules. Code which implements virtual memory areas (to allow device memory to be mapped into user space, for example) usually provides a `nopage()` function to handle page faults. The prototype for that function in 2.4.x and 2.6.0 is: 

```
struct page *(*nopage)(struct vm_area_struct *area, 
                               unsigned long address, 
    			   int unused);
```

As of 2.6.1, the `unused` argument is no longer unused, and the prototype has changed to: 

```
struct page *(*nopage)(struct vm_area_struct *area, 
    	                   unsigned long address, 
    			   int *type);
```

The `type` argument is now used to return the type of the page fault; `VM_FAULT_MINOR` would indicate a minor fault - one where the page was in memory, and all that was needed was a page table fixup. A return of `VM_FAULT_MAJOR` would, instead, indicate that the page had to be fetched from disk. Driver code using `nopage()` to implement a device mapping would probably return `VM_FAULT_MINOR`. In-tree code checks whether `type` is `NULL` before assigning the fault type; other users would be well advised to do the same. 

Making module code compile cleanly will require changing the prototype of the `nopage()` function, of course. 

As always, the [Driver Porting Series](<https://lwn.net/Articles/driver-porting/>) has been updated to reflect these changes.
