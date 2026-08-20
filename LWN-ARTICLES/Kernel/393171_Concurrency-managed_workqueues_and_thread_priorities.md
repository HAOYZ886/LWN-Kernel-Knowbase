---
title: "Concurrency-managed workqueues and thread priorities"
url: https://lwn.net/Articles/393171/
date: "June 22, 2010"
category: Kernel threads; Workqueues
author: "By Jonathan Corbet June 22, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 22, 2010

The original workqueue code found its way into the mainline without a great deal of discussion or debate; it was a clear improvement over what came before. Tejun Heo's [concurrency-managed workqueues](<http://lwn.net/Articles/355700/>) (CMWQ) rework has the potential to be a significant improvement as well, but its path toward merging has not been so smooth. The [fifth iteration of the patch set](<http://lwn.net/Articles/392037/>) is currently under discussion. While a number of concerns have been addressed, others have come out of the woodwork to replace them. 

The CMWQ work is intended to address a number of problems with current kernel workqueues. At the top of the list is the proliferation of kernel threads; current workqueues can, on a large system, run the kernel out of process IDs before user space ever gets a chance to run. Despite all these threads, current workqueues are not particularly good at keeping the system busy; workqueues may contain a backlog of work while the CPU sits idle. Workqueues can also be subject to deadlocks if locking is not handled very carefully. As a result, the kernel has grown a number of workarounds and some competing deferred-work mechanisms. 

To resolve these problems, the CMWQ code maintains a set of worker threads on each processor; these threads are shared between workqueues, so the system is not overrun with workqueue-specific threads. The special scheduler class once used by CMWQ is long gone, but the code still has hooks into the scheduler which it can use to track which worker threads are actually executing at any given time. If all workqueue threads on a CPU have blocked waiting on some resource, and if there is queued work to do, the CMWQ code will kick off a new thread to work on it. The CMWQ code can run multiple jobs from the same CPU concurrently - something the current workqueue code will not do. In this way, the CPU is always kept busy as long as there is work to be done. 

The first complaint that came back this time was that many developers had long since forgotten what CMWQ was all about, and Tejun had not put that information into the patch series introduction. He made up for that with [an overview document](<https://lwn.net/Articles/393172/>) explaining the basics of the code. That led quickly to a new complaint: the lack of dedicated worker threads means that it is no longer possible to change the scheduling behavior of specific workqueues. 

There were two variants of this complaint. Daniel Walker [lamented](<https://lwn.net/Articles/393173/>) the loss of the ability to change the priority of workqueue threads from user space. Tejun has firmly denied that this is a useful thing to be able to do, and Daniel has not, yet, shown an example of where it would be desirable. Andrew Morton, instead, [worries](<https://lwn.net/Articles/393174/>) about being able to change scheduling behavior from within the kernel; that is something that at least one driver does now. He [might be willing](<https://lwn.net/Articles/393175/>) to let this capability go, but he's not happy about it: 

Oh well. Kernel threads should not be running with RT policy anyway. RT is a userspace feature, and whenever a kernel thread uses RT it degrades userspace RT qos. But I expect that using RT in kernel threads is sometimes the best tradeoff, so let's not pretend that we're getting something for nothing here! 

Tejun's reply to this concern takes a couple of forms. One is that workqueues are intended for general-purpose asynchronous work, and that is how almost all callers use it. It would be better, he says, to make special mechanisms for situations where they are really needed. To that end, he has posted a simple [kthread_worker API](<http://lwn.net/Articles/392783/>) which can be used for the creation of special-purpose worker threads. Essentially, one starts by setting up a `kthread_worker` structure: 

```
DEFINE_KTHREAD_WORKER(worker);
        /* ... or ... */
        struct kthread_worker worker;
        init_kthread_worker(&worker);
```

Then, a kernel thread should be set up using the (existing) `kthread_create()` or `kthread_run()` utilities, but passing a pointer to `kthread_worker_fn()` as the actual function to run: 

```
struct task_struct thread;
    
        thread = kthread_run(kthread_worker_fn, &worker, "name" ...);
```

Thereafter, it's just a matter of filling in `kthread_work` structures with actual work to be done and queueing them with: 

```
bool queue_kthread_work(struct kthread_worker *worker,
                                struct kthread_work *work);
```

So far, there has been no real commentary on this patch. 

The other thing which could be done is to associate attributes like priority and CPU affinity with the work to be done instead of with the thread doing the work. That would require expanding the workqueue API to allow this information to be specified; the CMWQ code would then tweak worker threads accordingly when passing jobs to them. At this point, though, it's not clear that there is enough need for this feature to justify the added complexity that it would require. 

The CMWQ code certainly adds a bit of complexity already, though it makes up for some of that by replacing the [slow work](<http://lwn.net/Articles/329464/>) and [asynchronous function call](<http://lwn.net/Articles/314808/>) mechanisms. Tejun is hoping to drop it into linux-next shortly, and, presumably, to get it merged for 2.6.36. Whether that will happen remains to be seen; core kernel changes can be hard, and this one may not, yet, have cleared its last hurdle.
