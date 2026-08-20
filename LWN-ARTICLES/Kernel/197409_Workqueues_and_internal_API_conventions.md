---
title: Workqueues and internal API conventions
url: https://lwn.net/Articles/197409/
date: "August 29, 2006"
category: Workqueues
author: 
---

The internal kernel API has developed a number of conventions over the years. One of the most prevalent has to do with the return values from functions. In many cases, a function will return zero as an indicator of success, or a negative error code on failure. This convention goes against the normal C conventions for boolean values - a "false" value means that everything is OK. But it reflects the fact that, while all happy functions are alike, every unhappy function is unhappy in its own way. It is useful to be able to return a variety of error codes. 

There are exceptions to this convention, however. One of the more famous is `copy_to_user()` and `copy_from_user()`, both of which will, on failure, return the number of bytes which were not copied. Back in 2002, Rusty Russell audited 5500 calls to these functions and determined that 415 of them interpreted the return value incorrectly. He proposed changing the interface to match the kernel's conventions, but had no success. See [the May 23, 2002 LWN Kernel Page](<http://lwn.net/2002/0523/kernel.php3>) for more on this episode. 

More recently, Alan Stern has [been burned by the workqueue interface](<https://lwn.net/Articles/197410/>). Functions like `queue_work()` return a "normal" boolean value - zero on failure, non-zero if the requested work was actually queued. Alan suggested that these functions should be changed, and offered to fix up all in-tree callers in the process. The answer he got back was that fixing the return code would be a good thing, but that the name of the functions should be changed at the same time. Otherwise out-of-tree code could misinterpret the new return value with no indication to the programmer. 

The resulting [patch](<http://lwn.net/Articles/197318/>) does just that. With this patch, the functions for adding work to an arbitrary workqueue become: 

```
int add_work_to_q(struct workqueue_struct *queue, 
                          struct work_struct *work);
        int add_delayed_work_to_q(struct workqueue_struct *queue,
                                  struct work_struct *work,
    			      unsigned long delay);
        int add_delayed_work_to_q_on(int cpu,
                                     struct workqueue_struct *queue,
    				 struct work_struct *work,
    				 unsigned long delay);
```

As expected, these functions return zero on success and a negative error code (`-EBUSY`) on failure. The return code makes sense because the only reason for the operation to fail in current code is if the given `work_struct` is already on a workqueue. 

Similar changes have been made to the functions which operate on the generic, shared workqueue (`schedule_work()` and friends). They are now: 

```
int add_work(struct work_struct *work);
        int add_delayed_work(struct work_struct *work, unsigned long delay);
        int add_delayed_work_on(int cpu, struct work_struct *work,
                                unsigned long delay);
```

In all each case, wrapper functions with the old names have been provided so that out-of-tree code which has not been updated will not break. Most of the time, anyway. It seems that most in-tree callers never bothered to check the return value from these functions in the first place, and Alan has concluded that out-of-tree callers will be the same. So the new version of the old functions are declared as `void`, returning no value at all. Instead, they log a warning when an operation fails. As a result of this change, code which actually checks the return value will fail to compile, and, presumably, the author will update it to the new functions. Everything else will continue to run as it always did. 

Alan has also proposed an addition to the kernel coding style document. It reads (in part): 

If the name of a function is an action or an imperative command, the function should return an error-code integer. If the name is a predicate, the function should return a "succeeded" boolean. 

There does not seem to be much disagreement over this proposal, so that is likely to be how things go. This convention is still not likely to extend to `copy_to_user()` and `copy_from_user()`, however.
