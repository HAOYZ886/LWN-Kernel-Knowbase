---
title: Deadline scheduler part 2 — details and usage
url: https://lwn.net/Articles/743946/
date: "January 19, 2018"
category: "Realtime-Deadline scheduling; Scheduler-Deadline scheduling"
author: "January 19, 2018 This article was contributed by Daniel Bristot de Oliveira"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

January 19, 2018

This article was contributed by Daniel Bristot de Oliveira

Linux’s deadline scheduler is a global early deadline first scheduler for sporadic tasks with constrained deadlines. These terms were defined in [the first part of this series](<https://lwn.net/Articles/743740/>). In this installment, the details of the Linux deadline scheduler and how it can be used will be examined. 

The deadline scheduler prioritizes the tasks according to the task’s job deadline: the earliest absolute deadline first. For a system with _M_ processors, the _M_ earliest deadline jobs will be selected to run on the _M_ processors. 

The Linux deadline scheduler also implements the constant bandwidth server (CBS) algorithm, which is a resource-reservation protocol. CBS is used to guarantee that each task will receive its full run time during every period. At every activation of a task, the CBS replenishes the task’s run time. As the job runs, it consumes that time; if the task runs out, it will be throttled and descheduled. In this case, the task will be able to run only after the next replenishment at the beginning of the next period. Therefore, CBS is used to both guarantee each task’s CPU time based on its timing requirements and to prevent a misbehaving task from running for more than its run time and causing problems to other jobs. 

In order to avoid overloading the system with deadline tasks, the deadline scheduler implements an acceptance test, which is done every time a task is configured to run with the deadline scheduler. This test guarantees that deadline tasks will not use more than the maximum amount of the system's CPU time, which is specified using the `kernel.sched_rt_runtime_us` and `kernel.sched_rt_period_us` sysctl knobs. The default values are 950000 and 1000000, respectively, limiting realtime tasks to 950,000µs of CPU time every 1s of wall-clock time. For a single-core system, this test is both necessary and sufficient. It means that the acceptance of a task guarantees that the task will be able to use all the run time allocated to it before its deadline. 

However, it is worth noting that this acceptance test is necessary, but _not_ sufficient, for global scheduling on multiprocessor systems. As Dhall’s effect (described in the first part of this series) shows, the global deadline scheduler acceptance task is unable to schedule the task set even though there is CPU time available. Hence, the current acceptance test does not guarantee that, once accepted, the tasks will be able to use all the assigned run time before their deadlines. The best the current acceptance task can guarantee is bounded tardiness, which is a good guarantee for soft real-time systems. If the user wants to guarantee that all tasks will meet their deadlines, the user has to either use a partitioned approach or to use a necessary and sufficient acceptance test, defined by: 

> Σ(WCETi / Pi) <= M - (M - 1) x Umax 

Or, expressed in words: the sum of the run time/period of each task should be less than or equal to the number of processors, minus the largest utilization multiplied by the number of processors minus one. It turns out that, the bigger Umax is, the less load the system is able to handle. 

In the presence of tasks with a big utilization, one good strategy is to partition the system and isolate some high-load tasks in a way that allows the small-utilization tasks to be globally scheduled on a different set of CPUs. Currently, the deadline scheduler does not enable the user to set the affinity of a thread, but it is possible to partition a system using control-group cpusets. 

For example, consider a system with eight CPUs. One big task has a utilization close to 90% of one CPU, while a set of many other tasks have a lower utilization. In this environment, one recommended setup would be to isolate CPU0 to run the high-utilization task while allowing the other tasks to run in the remaining CPUs. To configure this environment, the user must follow the following steps: 

  1. Enter in the cpuset directory and create two cpusets: 

```
# cd /sys/fs/cgroup/cpuset/
             # mkdir cluster
             # mkdir partition
```

  2. Disable load balancing in the root cpuset to create two new root domains in the CPU sets: 

```
# echo 0 > cpuset.sched_load_balance
```

  3. Enter the directory for the cluster cpuset, set the CPUs available to 1-7, the memory node the set should run in (in this case the system is not NUMA, so it is always node zero), and set the cpuset to the exclusive mode. 

```
# cd cluster/
             # echo 1-7 > cpuset.cpus
             # echo 0 > cpuset.mems
             # echo 1 > cpuset.cpu_exclusive
```

  4. Move all tasks to this CPU set 

```
# ps -eLo lwp | while read thread; do echo $thread > tasks ; done
```

  5. Then it is possible to start deadline tasks in this cpuset. 
  6. Configure the partition cpuset: 

```
# cd ../partition/
             # echo 1 > cpuset.cpu_exclusive 
             # echo 0 > cpuset.mems 
             # echo 0 > cpuset.cpus
```

  7. Finally move the shell to the partition cpuset. 

```
# echo $$ > tasks
```

  8. The final step is to run the deadline workload. 

With this setup, the task isolated in the partitioned cpuset will not interfere with the tasks in the cluster cpuset, increasing the system’s maximum load while meeting the deadline of real-time tasks. 

#### The developer’s perspective

There are three ways to use the deadline scheduler: as constant bandwidth server, as a periodic/sporadic server waiting for an event, or with a periodic task waiting for replenishment. The most basic parameter for the sched deadline is the period, which defines how often a task is activated. When a task does not have an activation pattern, it is possible to use the deadline scheduler in an aperiodic mode by using only the CBS features. 

In the aperiodic case, the best thing the user can do is to estimate how much CPU time a task needs in a given period of time to accomplish the expected result. For instance, if one task needs 200ms each second to accomplish its work, run time would be 200,000,000ns and the period would be 1,000,000,000ns. The [`sched_setattr()`](<http://man7.org/linux/man-pages/man2/sched_setattr.2.html>) system call is used to set the deadline-scheduling parameters. The following code is a simple example of how to set the mentioned parameters in an application: 

```
int main (int argc, char **argv)
        {
            int ret;
            int flags = 0;
            struct sched_attr attr;
    
            memset(&attr, 0, sizeof(attr)); 
            attr.size = sizeof(attr);
    
            /* This creates a 200ms / 1s reservation */
            attr.sched_policy   = SCHED_DEADLINE;
            attr.sched_runtime  =  200000000;
            attr.sched_deadline = attr.sched_period = 1000000000;
    
            ret = sched_setattr(0, &attr, flags);
            if (ret < 0) {
                perror("sched_setattr failed to set the priorities");
                exit(-1);
            }
    
            do_the_computation_without_blocking();
            exit(0);
    }
```

In the aperiodic case, the task does not need to know when a period starts, and so the task just needs to run, knowing that the scheduler will throttle the task after it has consumed the specified run time. 

Another use case is to implement a periodic task which starts to run at every periodic run-time replenishment, runs until it finishes its processing, then goes to sleep until the next activation. Using the parameters from the previous example, the following code sample uses the [`sched_yield()`](<http://man7.org/linux/man-pages/man2/sched_yield.2.html>) system call to notify the scheduler of end of the current activation. The task will be awakened by the next run-time replenishment. Note that the semantics of `sched_yield()` are a bit different for deadline tasks; they will not be scheduled again until the run-time replenishment happens. 

Code working in this mode would look like the example above, except that the actual computation looks like: 

```
for(;;) {
                do_the_computation();
                /* 
    	     * Notify the scheduler the end of the computation
                 * This syscall will block until the next replenishment
                 */
    	    sched_yield();
            }
```

It is worth noting that the computation must finish within the given run time. If the task does not finish, it will be throttled by the CBS algorithm. 

The most common realtime use case for the realtime task is to wait for an external event to take place. In this case, the task waits in a blocking system call. This system call will wake up the real-time task with, at least, a minimum interval between each activation. That is, it is a sporadic task. Once activated, the task will do the computation and provide the response. Once the task provides the output, the task goes to sleep by blocking waiting for the next event. 

```
for(;;) {
                /* 
    	     * Block in a blocking system call waiting for a data
                 * to be processed.
                 */
                process_the_data();
                produce_the_result()
    	    block_waiting_for_the_next_event();
            }
```

#### Conclusion

The deadline scheduler is able to provide guarantees for realtime tasks based only in the task’s timing constraints. Although global multi-core scheduling faces Dhall’s effect, it is possible to configure the system to achieve a high load utilization using cpusets as a method to partition the systems. Developers can also benefit from the deadline scheduler by designing their application to interact with the scheduler, simplifying the control of the timing behavior of the task. 

The deadline scheduler tasks have a higher priority than realtime scheduler tasks. That means that even the highest fixed-priority task will be delayed by deadline tasks. Thus, deadline tasks do not need to consider interference from realtime tasks, but realtime tasks must consider interference from deadline tasks. 

The deadline scheduler and the PREEMPT_RT patch play different roles in improving Linux’s realtime features. While the deadline scheduler allows scheduling tasks in a more predictable way, the PREEMPT_RT patch set improves the kernel by reducing and limiting the amount of time a lower-priority task can delay the execution of a realtime task. It works by reducing the amount of the time a processor runs with preemption and IRQs disabled and the amount of time in which a lower-priority task can delay the execution of a task by holding a lock. 

For example, as a realtime task can suffer an activation latency higher than 5ms when running in a non-realtime kernel, it is that this kernel cannot handle deadline tasks with deadlines shorter than 5ms. In contrast, the realtime kernel provides a guarantee, on well tuned and certified hardware, of not delaying the start of the highest priority task by more than 150µs, thus it is possible to handle realtime tasks with deadlines much shorter than 5ms. You can find more about the realtime kernel [here](<http://developerblog.redhat.com/?p=425603&preview_id=425603&preview_nonce=28c03def3d&post_format=standard&preview=true>). 

Acknowledgment: this series of articles was reviewed and improved with comments from Clark Williams, Beth Uptagrafft, Arnaldo Carvalho de Melo, Luis Claudio R. Gonçalves, Oleksandr Natalenko, Jiri Kastner and Tommaso Cucinotta.
