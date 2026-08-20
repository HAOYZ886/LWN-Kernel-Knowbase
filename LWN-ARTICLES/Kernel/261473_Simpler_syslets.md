---
title: Simpler syslets
url: https://lwn.net/Articles/261473/
date: "December 10, 2007"
category: Syslets
author: "By Jonathan Corbet December 10, 2007"
---

By **Jonathan Corbet**  
December 10, 2007

Syslets are a proposed mechanism which would allow any system call to be invoked in an asynchronous manner; this technique promises a more comprehensive and simpler asynchronous I/O mechanism and much more - once all of the pesky little details can be worked out. A while back, Zach Brown let it be known that he had taken over the ongoing development of the syslets patch set; things have been relatively quiet since then. But Zach has just returned with [a new syslets patch](<http://lwn.net/Articles/261325/>) which shows where this idea is going. 

This version of the patch removes much of the functionality seen in previous postings. The ability to load simple programs into the kernel for asynchronous execution is now gone, as is the "threadlet" mechanism for asynchronous execution of user-space functions. Instead, syslets have gone back to their roots: a mechanism for running a single system call without blocking. 

As had been foreshadowed in other discussions, syslets now use the [`indirect()`](<http://lwn.net/Articles/259068/>) system call mechanism. An application wanting to perform an asynchronous system call fills in a `syslet_args` structure describing how the asynchronous execution is to be handled; the application then calls `indirect()` to make it happen. If the system call can run without blocking, `indirect()` simply returns with the final status. If blocking is required, the kernel will (as with previous versions of this patch) return to user space in a separate process while the original process waits for things to complete. Upon completion, the final status is stored in user-space memory and the application is notified in an interesting way. 

The `syslet_args` structure looks like this: 

```
struct syslet_args {
    	u64 completion_ring_ptr;
    	u64 caller_data;
    	struct syslet_frame frame;
        };
```

The `completion_ring_pointer` field contains a pointer to a circular buffer stored in user space. The head of the buffer is defined this way: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

```
struct syslet_ring {
    	u32 kernel_head;
    	u32 user_tail;
    	u32 elements;
    	u32 wait_group;
    	struct syslet_completion comp[0];
        };
```

Here, `kernel_head` is the index of the next completion ring entry to be filled in by the kernel, and `user_tail` is the next entry to be consumed by the application. If the two are equal, the ring is empty. The `elements` field says how many entries can be stored in the ring; it must be a power of two. The kernel uses `wait_group` as a way of locating a wait queue internally when the application waits on syslet completion; your editor suspects that this part of the API may not survive into the final version. 

Finally, the completion status values themselves live in the array of `syslet_completion` structures, which look like this: 

```
struct syslet_completion {
    	u64 status;
    	u64 caller_data;
        };
```

When a syslet completes, the final return code is stored in `status`, while the `caller_data` field is set with the value provided in the field by the same name in the `syslet_args` structure when the syslet was first started. 

There is one field of `syslet_args` which has not been discussed yet: `frame`. The definition of this structure is architecture-dependent; for the x86 architecture it is: 

```
struct syslet_frame {
    	u64 ip;
    	u64 sp;
        };
```

These values are used when the syslet completes. After the kernel stores the completion status in the ring buffer, it will call the function whose address is stored in `ip`, using the stack pointer found in `sp`. This call serves as a sort of instant, asynchronous notification to the application that the syslet is done. It's worth noting that this call is performed in the original process - the one in which the syslet was executed - rather than in the new process used to return to user space when the syslet blocked. This function also has nothing to return to, so, after doing its job, it should simply exit. 

So, to review, here is how a user-space application will use syslets to call a system call asynchronously: 

  * The completion ring is established and initialized in user space. 

  * A stack is allocated for the notification function, and the `syslet_args` structure is filled in with the relevant information. 

  * A call is made to `indirect()` to get the syslet going. 

  * If the system call of interest is able to complete without blocking, the return value is passed directly back to user space from `indirect()` and the call is complete. 

  * Otherwise, once the system call blocks, execution switches to a new process which returns to user space. An `ESYSLETPENDING` error is returned in this case. 

  * Once the system call completes, the kernel stores the return value in the completion ring and calls the notification function in the original process. 

Should the application wish to stop and wait for any outstanding syslets to complete, it can make use of a new system call: 

```
int syslet_ring_wait(struct syslet_ring *ring, unsigned long user_idx);
```

Here, `ring` is the pointer to the completion ring, and `user_idx` is the value of the `user_tail` index as seen by the process. Providing the tail as an argument to `syslet_ring_wait()` prevents problems with race conditions which might come about if a syslet completes after the application has decided to wait. This call will return once there is at least one completion in the ring. 

The real purpose of this set of patches is to try to nail down the user-space API for syslets; it is clear that there is still some work to be done. For example, there is no way, currently, for an application to use `indirect()` to simultaneously launch a syslet and (as was the original purpose for `indirect()`) provide additional arguments to the target system call. In fact, the means for determining which of the two is being done looks dangerously brittle. As Zach has already noted, the calling convention needs to be changed to make the syslet functionality and the addition of arguments orthogonal. 

There are a number of other questions which need to be answered - Zach has supplied a few of them with the patch. Interaction with `ptrace()` is unclear, resource management issues abound, and so on. Zach is clearly looking for feedback on these issues: 

I'm particularly interested in hearing from people who are trying to use syslets in their applications. This will involve awkward wrappers instead of glibc calls for now, and your machine may explode, but hopefully the chance to influence the design of syslets would make it worth the effort. 

So, the message is clear: anybody who is interested in how this interface will look would be well advised to pay attention to it now.
