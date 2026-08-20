---
title: "FUTEX + rwsem = SNAFU"
url: https://lwn.net/Articles/124747/
date: "February 23, 2005"
category: Futex; Race conditions; get user inatomic
author: 
---

The FUTEX code implements lightweight mutual exclusion primitives for user space. It is intended to be used in situations - such as multi-threaded programs - where mutual exclusion is needed, but where the implementation must be fast. Olof Johansson recently [stumbled across](<https://lwn.net/Articles/124754/>) a case where the FUTEX code can deadlock the system (thus failing the "fast" test) which shows how hard it can be to get concurrency issues right. 

One of the many locking primitives provided by the kernel is the reader-writer semaphore, or "rwsem". An rwsem can be obtained for either read or write access. Any number of readers will be allowed to hold the semaphore concurrently. Any thread which must change the protected data structures must, however, obtain the semaphore for write access. Only one writer is allowed at any given time, and no readers may be in the critical section while the writer is at work. 

If a thread tries to obtain an rwsem for write access, and that semaphore is currently held (by somebody else) for read access, the writer will be put to sleep. Once the writer gets in line, however, no more readers will be allowed in. Once the existing readers have gotten out of the way, the writer will be allowed to proceed. The queued readers will only wake up after the writer is done. This implementation makes rwsems fair, in that readers cannot starve writers indefinitely. It also makes certain types of subtle faults possible, however. 

If a process might have to wait on a FUTEX, the kernel must obtain that process's memory map semaphore (`mmap_sem`). This semaphore, which is an rwsem, controls access to the internal FUTEX data structures; it is taken for read access. The kernel must also query the value of the FUTEX itself, which is done through a call to `get_user()`. Should that access generate a page fault, the fault handler will obtain `mmap_sem` for read access a second time. This double access works just fine; the second `down_read()` call simply looks like another reader, which can run concurrently with the first. 

Life gets complicated, however, when other processes share the same address space. Since the FUTEX mechanism is aimed at threads, this is a situation which comes about frequently. Consider the following series of events: 

Thread 1| Thread 2  
---|---  
Call `sys_futex()`|   
`down_read(&current->mm->mmap_sem);` |   
| call `mmap()`  
| `down_write(&current->mm->mmap_sem);`  
|  (goes to sleep)  
call get_user()|   
(everything comes to a halt)  
  
When the second process calls `mmap()`, it must obtain `mmap_sem` for write access. Since the first process is already a reader, the `down_write()` call is queued and the process is put to sleep. When the first process makes its `get_user()` call, it tries to obtain the rwsem for read access for the second time. Since there is now a writer waiting, however, the first process also is put to sleep. Since the first process is the one holding the initial read lock, this situation will never resolve itself; it is a deadlock. This particular type of deadlock is nasty in that it requires a race condition to become visible; things usually just work. 

Several possible solutions have been proposed. The rwsem "lock depth" could be [explicitly tracked](<https://lwn.net/Articles/124757/>) so that a second attempt to obtain read access simply implements a counter and does not sleep. Processes holding `mmap_sem` could be marked with a special `PF_MMAP_SEM` flag; the page fault code would see that flag, realize that the semaphore is already held, and not take it again. Olof's initial report included a patch which tries to explicitly fault in the page before taking the semaphore so that the `get_user()` call would not generate a fault. 

The solution which will eventually be adopted will likely take a different approach, however. Conventional wisdom has long said that functions like `get_user()` cannot be called in atomic context (in an interrupt handler or when a spinlock is held), since they might sleep. In fact, if the user-space access functions generate a page fault in atomic context, the fault handler simply refuses to bring in the page and the function returns an error code. So the solution, first [suggested](<https://lwn.net/Articles/124758/>) by Linus, is to put the process into an atomic mode (by calling `inc_preempt_count()`) just before the `get_user()` call. If `get_user()` fails, the page must be faulted in. So the `mmap_sem` is dropped, the page is explicitly faulted, and the whole process starts over again. 

As often happens, the full solution turned out to be a bit more complicated than initially thought. So Olof put together [a patch](<https://lwn.net/Articles/124725/>) implementing a new user-space access function: 

```
int get_user_inatomic(value, user_pointer);
```

This function is atomic; it may succeed or fail, but it will always return without sleeping. Like `get_user()`, it is implemented as a macro which tries to do the right thing regardless of the data type of the value to be fetched. That implementation drew [a complaint](<https://lwn.net/Articles/124759/>) from one developer, who would rather see new interfaces done in a more strongly-typed manner. So the details of the patch that eventually gets merged (presumably after 2.6.11) may change, but it will likely follow this approach.
