---
title: "Single-threaded workqueues"
url: https://lwn.net/Articles/82525/
date: "April 28, 2004"
category: Workqueues
author: 
---

The [workqueue](<https://lwn.net/Articles/23634/>) mechanism is the 2.6 kernel's replacement for task queues; a workqueue allows kernel code to defer work until some time in the future. Tasks submitted to work queues are run in the context of a special process, so they can sleep if need be. Workqueues go out of their way to keep work on the same processor by a dedicated worker thread for each processor on the system. 

For many applications, one process per CPU is far more than is needed; a single worker process is plenty. There is a shared, generic workqueue which can be used in many of these situations. In others, however, use of that queue is not appropriate; perhaps the code in question performs long sleeps, or it may deadlock with another use of that queue. In these cases, there has been no alternative to paying the cost of all those worker threads. 

As of 2.6.6, thanks to Rusty Russell, there will be a new function for creating workqueues: 

```
struct workqueue_struct *create_singlethread_workqueue(char *name);
```

As you might expect, this function creates a workqueue that relies on a single worker thread. Chances are, many of the current users of workqueues could switch over to the single-threaded variety.
