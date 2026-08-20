---
title: Kernel threads made easy
url: https://lwn.net/Articles/65178/
date: "January 6, 2004"
category: Kernel threads
author: 
---

It is fairly common for kernel code to create lightweight processes - kernel threads - which perform a certain task asynchronously. To see these threads, run `ps ax` on a 2.6 kernel and note all of the processes in [square brackets] at the beginning of the listing. The code which sets up these threads has tended to be reimplemented every time a new thread is needed, however, and certain tasks (ensuring that the environment is clean, for example) are not always handled well. The current kernel also does not easily allow the creator of a kernel thread to control the behavior of that thread. 

Rusty Russell encountered even more trouble as he was doing his "hotplug CPU" work: when processors can come and go, their associated kernel threads must be started or stopped at arbitrary times. To make his life easier, he [implemented a new set of kernel thread primitives](<https://lwn.net/Articles/65434/>) which simplify the task greatly. 

Using the new mechanism, the first step in creating a kernel thread is to define a "thread function" which will contain the code to be executed; it has a prototype like: 

```
int thread_function(void *data);
```

The function will be called repeatedly (if need be) by the kthread code; it can perform whatever task it is designated to do, sleeping when necessary. This function should, however, check its signal status and return if any signals are pending. 

A kernel thread is created with: 

```
struct task_struct *kthread_create(int (*threadfn)(void *data),
                                           void *data,
    				       const char *namefmt, ...);
```

The `data` argument will simply be passed to the thread function. A standard `printk()`-style formatted string can be used to name the thread. The thread will not start running immediately; to get the thread to run, pass the `task_struct` pointer returned by `kthread_create()` to `wake_up_process()`. 

There is also a convenience function which creates and starts the thread: 

```
struct task_struct *kthread_run(int (*threadfn)(void *data),
                                        void *data,
    				    const char *namefmt, ...);
```

Once started, the thread will run until it explicitly calls `do_exit()`, or until somebody calls `kthread_stop()`: 

```
int kthread_stop(struct task_struct *thread);
```

`kthread_stop()` works by sending a signal to the thread. As a result, the thread function will not be interrupted in the middle of some important task. But, if the thread function never returns and does not check for signals, it will never actually stop. 

Kernel threads are often created to run on a particular processor. To achieve this effect, call `kthread_bind()` after the thread is created: 

```
void kthread_bind(struct task_struct *thread, int cpu);
```

Rusty's patch includes a set of changes converting a number of kernel thread users over to the new infrastructure. There has been a fair amount of discussion of the kthread patches, which has resulted in some significant changes. Whether this code will get into the 2.6 kernel remains to be seen, however.
