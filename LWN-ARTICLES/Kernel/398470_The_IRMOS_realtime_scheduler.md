---
title: The IRMOS realtime scheduler
url: https://lwn.net/Articles/398470/
date: "August 3, 2010"
category: "Realtime-Deadline scheduling; Scheduler-Deadline scheduling"
author: "August 3, 2010 By T. Cucinotta and F. Checconi"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

August 3, 2010

By [T. Cucinotta](<http://retis.sssup.it/~tommaso>) and [F. Checconi](<http://retis.sssup.it/~fabio>)

In the context of the [IRMOS European Project](<http://www.irmosproject.eu>) (Interactive Real-Time Applications on Service-Oriented Infrastructures), a new realtime scheduler for Linux has been developed by the [Real-Time Systems Laboratory](<http://retis.sssup.it>) of [Scuola Superiore Sant'Anna](<http://www.sssup.it>) in Pisa. The purpose of this article is to provide a general overview of this new scheduler, describe its features and how it can be practically used, provide a few details about the implemented algorithms, and gathering feedback by the community about possible improvements. 

The IRMOS realtime scheduler (a.k.a., EDF throttling or realtime throttling), allows the administrator to reserve a "slice" of the processing capability of a system for a group of Linux tasks. It is based on a direct modification of the POSIX realtime scheduling class within the Linux kernel, and in particular, to the throttling mechanism already built into the kernel for realtime tasks. Basically, the realtime throttling mechanism is changed from a mechanism that _exclusively limits_ the computation power granted to groups of realtime tasks, to one that provides them with both a limit and precise scheduling guarantees (in terms of a guaranteed runtime every period, on each of the available CPUs). Also, it has been designed from scratch with SMP support in mind, and it implements a hierarchical scheduling policy based on both deadlines and priorities. Specifically, POSIX fixed priority (FP) realtime scheduling is nested inside EDF (Earliest Deadline First) scheduling. 

The IRMOS realtime scheduler allows for the provisioning of scheduling guarantees to individual task groups. This provisioning is done by specifying two scheduling parameters: a _budget_ Q and a _period_ P. The tasks in the group are entitled to run on each of the CPUs (processor, or cores when present) available on the platform, for Q time units every period of P time units. This constitutes a scheduling guarantee and a limitation at the same time. 

> ![\[Partitions\]](https://static.lwn.net/images/ns/kernel/irmos/cpu-partitions.png)

For example, on a single-CPU system, a single task attached to a reservation of 10ms every 100ms is guaranteed to be scheduled on the CPU for 10ms every 100ms. If the task tries to execute for more than 10ms, then the scheduler removes it from the run queue until the next period, at which point its budget is refilled. So if the system has no other ready tasks to schedule, then the CPU goes idle in this time. 

Note that periods of different reservations may be specified independently from each other, and the above guarantee is still valid. 

The EDF-based scheduler applies a simple admission control rule that decides whether a new reservation may be accepted; it works by ensuring that the sum of the utilizations (budget over period) of all the reservations is less than or equal to the maximum configured share assigned to realtime tasks. This limit may be configured through the `cpu.rt_runtime_us` and `cpu.rt_period_us` entries of the root-level cgroup filesystem (see the tutorial below). Theoretically, the EDF-based scheduling algorithm allows for full utilization of each CPU by realtime tasks, provided that those tasks can be properly partitioned across the CPUs. However, from a practical perspective, this is far from being a desirable working condition. 

#### The CBS: EDF-based Scheduling and Temporal Isolation

The deadline-based part of the IRMOS scheduler is an implementation of a hard-reservation variant of the Constant Bandwidth Server (CBS) algorithm, described in "[Integrating Multimedia Applications in Hard Realtime Systems](<http://portal.acm.org/citation.cfm?id=829047>)" by Abeni and Buttazzo. Let us take a peek at how this works, focusing on a single-CPU system, where independent reservations are scheduled. 

For each reservation, in addition to the configured budget and period values, the kernel manages a _current budget_ and a _current deadline_. Reservations are scheduled on each CPU depending on their current deadlines, using the earliest deadline first algorithm. The first time a reservation is activated, the current deadline is initialized to the activation time plus the configured period, and the current budget is set equal to the configured budget. Each time any task in the reservation is scheduled for some time on the CPU, the current budget is decreased by the same time value. Once the current budget goes to zero (it may also become negative due to non-interruptible kernel sections -- see below), the reservation is suspended (throttled) till the next activation period, when the current budget is refilled again to the configured value, and the deadline is moved forward by a value equal to the period. 

A sample schedule is shown in the diagram below, for two tasks with reservations of `5ms` every `9ms` and `2ms` every `6ms`, respectively, for an overall utilization of about `88.9%`. 

> ![\[Scheduler diagram\]](https://static.lwn.net/images/ns/kernel/irmos/sched-edf.png)

However, this is not enough yet, in order to guarantee _temporal isolation_ among independent reservations. If one of the reserved tasks tried to consume more CPU than allocated, then it could potentially cause a deadline miss for another task which is, instead, behaving according to the declared parameters: 

> ![\[Deadline miss\]](https://static.lwn.net/images/ns/kernel/irmos/sched-edf-noenforce.png)

To avoid this problem, the offending task is suspended by the kernel until the next period: 

> ![\[Period enforcement\]](https://static.lwn.net/images/ns/kernel/irmos/sched-edf-enforce.png)

Furthermore, whenever the reservations become non-runnable (e.g., all of the attached tasks block, then someone wakes up later) in a way that does not fit into the classical periodic activation pattern, we have another potential problem. For example, if a reservation becomes runnable too close to its current deadline, and the current deadline is not changed, then it will be selected by the EDF scheduler as the most urgent one to schedule, causing a potentially arbitrarily long delay to any other reservation on the same CPU: 

> ![\[Blocking tasks\]](https://static.lwn.net/images/ns/kernel/irmos/sched-edf-block.png)

To mitigate this problem, when a reservation wakes up as a consequence of a task unblocking itself, the scheduler may behave in one of two ways: if a relatively small time has elapsed since the process blocked, then the kernel keeps the same deadline and budget for the reservation. However, if an excessive amount of time has elapsed, then the kernel "resets" the deadline to the current time plus the reservation period, and the current budget to the allocated reservation budget: 

> ![\[Shifting deadlines\]](https://static.lwn.net/images/ns/kernel/irmos/sched-edf-block-cbs.png)

More specifically, if the remaining budget divided by the time left until the current deadline does not exceed the bandwidth allocated to the task (equal to the configured reservation budget over the reservation period), then the current deadline and budget are preserved, otherwise they are reset. See the paper on the [CBS algorithm](<http://portal.acm.org/citation.cfm?id=829047>) for details and a formal proof that this rule ensures temporal isolation among reserved independent task groups, regardless of their actual temporal behavior. 

The above described mechanism has also the desirable property of self-synchronizing the scheduler with the temporal behavior of realtime tasks. In fact, when a reservation is attached to a single classical periodic realtime task, as soon as it wakes up in response to some (almost) periodic event, the scheduler will probably move its current deadline back to the wake-up time plus the period. On the other hand, such action is not usually done for very short sleeps of the task during its main execution body, e.g., in case it blocks on short critical sections for sharing data with other tasks of the same application. 

#### Hierarchical Scheduling

![\[Hierarchical scheduling\]](https://static.lwn.net/images/ns/kernel/irmos/hier-sched.png) The IRMOS scheduler features hierarchical scheduling, mixing both deadline-based and priority-based scheduling. Specifically, POSIX priority-based realtime scheduling is nested inside EDF-based scheduling. The situation is depicted in the figure to the right. 

When a reservation is selected to run by the partitioned EDF-based scheduler, a global POSIX priority-based scheduling policy decides what tasks belonging to that reservation will actually run on each CPU. If there are M CPUs, at most the M tasks with the highest priority (among the ones belonging to the reservation group) are the ones which actually run. The system performs admission control over admitted reserved groups, so that the overall system capacity may be properly partitioned among concurrently running activities in the system, without overloading it. Also, the scheduler has a hierarchical configuration capability, by which it is possible to define groups and nested subgroups of realtime tasks with given scheduling parameters. 

Further details about the IRMOS realtime scheduler are omitted for the sake of brevity, however the interested reader can refer to the paper "[Hierarchical Multiprocessor CPU Reservations for the Linux Kernel](<http://retis.sssup.it/~tommaso/eng/publications#L_OSPERT09-HMPR>)" describing the scheduler which appeared at [OSPERT 2009](<http://www.artist-embedded.org/artist/Overview,1750.html>). Any comments and feedback on the project by Linux users and developers is more than welcome. Authors can be contacted by using the [AQuoSA mailing lists](<https://sourceforge.net/mail/?group_id=152172>). 

#### Implementation Details

The IRMOS scheduler is implemented as a partitioned scheduling strategy: each reserved task group corresponds to a set of CBS reservations allocated (with identical parameters) on the CBS schedulers running independently on all of the available CPUs. The overall design of the current sched_rt implementation does not change; it still keeps one private run queue per CPU, and, thus, each CPU is scheduled independently of the others. 

That said, using EDF to schedule groups and a fixed priority (FP) scheme among the tasks of each group requires using a different representation for groups and tasks within the run queues, so a `sched_entity` represents only tasks within the group they belong to, and the EDF-related parameters (deadline, budget, period) are kept inside the `rt_rq` describing the actual run queue associated to each cgroup (note that the `rt_rq` is, at the same time, the data structure enqueued with EDF parameters into the run queue it belongs to, and the fixed-priority queue responsible, once selected, for the priority-based scheduling of its own tasks). 

The existing code represents groups of tasks using `struct task_group` objects; tasks are grouped on the basis of the cgroup they belong to, and each task group contains an array of per-processor run queues (`rt_rq`s). Tasks are represented by their own scheduling entity. The full hierarchy seen by the user is used internally only for admission control, as the scheduler itself enqueues all the `rt_rq`s based on their deadlines in the first-level run queue, and all the tasks are enqueued into the priority queue of the `rt_rq` they belong to. On each scheduling decision the (unthrottled) `rt_rq` with the smallest deadline is selected, and its highest-priority task is executed. When the tasks inside an `rt_rq` consume their assigned budget they will be throttled; their `rt_rq` is dequeued from the EDF run queue, and a timer is posted to recharge the run time, update the deadline, and requeue the `rt_rq` for the next period. 

Using the full hierarchical setup for admission control introduces an extra element of complexity in the interface, because, in general, for each group, the user needs to specify the overall bandwidth assigned to the group and to its child groups, as well as the bandwidth assigned to the tasks belonging to the group itself. This increases the number of parameters for each group from two to four. 

To avoid priority inversion problems, the scheduler uses, as the old throttling mechanism does, boosting; it lets groups with tasks inside critical sections run even if they should be throttled, charging them with the extra CPU time consumed only after they exit the critical section. From the implementation perspective this means that the EDF run queue also may contain boosted groups, which are scheduled only according to the highest priority among those of the tasks they contain; boosted `rt_rq`s take precedence over the other ones. 

#### Admission control and deadline guarantees

When considering realtime systems, we might be concerned about how, exactly, to exploit the described realtime scheduling policy in order to provide proper realtime guarantees to applications. In relatively simple cases, the answer is straightforward. For example, a classical periodic realtime task with a known worst case execution time (WCET) of C and minimum inter-arrival period of T, can be scheduled within a reservation with budget equal to C and reservation period equal to T and will not miss any deadlines. Also, the admission test in this case is the well-known Liu and Layland test for EDF realtime tasks (sum of utilizations must be less than or equal to 1). 

However, looking at realtime theory, one easily finds much more complex realtime task models, which include activation offsets, maximum blocking times, durations of critical sections accessing shared resources, etc. Also, real-world realtime applications are often complex multi-threaded applications (think of `vlc`) which are very far from behaving like foreseen by the "ideal" periodic or sporadic task model, and whose activation times are driven by disk I/O and networking instead of (or in addition to) timers. Furthermore, if the application is distributed, one has usually a distributed end-to-end deadline constraint to deal with, something out of reach for a kernel-level task scheduler. 

Under such a challenging scenario, it is still possible to schedule realtime applications with a simple policy based on the fundamental principle of temporal isolation, like the one being presented in this article, and provide the necessary guarantees. However, the admission test becomes complex, involving long and involved computations, thus prohibitive for the kernel. For a list of possible admission control tests for realtime applications scheduled with various policies (including EDF and FP), the reader can have a look at the proceedings of conferences dedicated to realtime scheduling, such as the [ Real-Time Systems Symposium (RTSS)](<http://www.rtss.org>), the [EUROMICRO Conference on Real-Time Systems (ECRTS)](<http://ecrts.eit.uni-kl.de/ecrts10>), the [IEEE Real-Time and Embedded Technology and Applications Symposium](<http://www.rtas.org/>), or others. 

This complexity is why, in the EDF-based scheduler described above, the realtime scheduling parameters communicated to the kernel are kept at the bare minimum and are used in a very simple admission control test. This approach does not try to guarantee that all admitted applications will meet their deadlines, but rather it aims to provide to each application a guaranteed share of the available underlying computing power, with a precise timing granularity. Whether this is sufficient or not for guaranteeing the performance of specific applications must be confirmed by other means, involving a proper design methodology and benchmarking process, possibly with the help of user-space middleware. 

#### A short tutorial

In order to try the IRMOS realtime scheduler, you can get the latest changes pushed on the [on-line git repository](<http://aquosa.git.sourceforge.net/git/gitweb.cgi?p=aquosa/linux-irmos;a=summary>) (currently corresponding to the `2.6.34-rc5` series) with: 

```
git clone git://aquosa.git.sourceforge.net/gitroot/aquosa/linux-irmos
```

or, for the `PREEMPT_RT` port: 

```
git clone git://aquosa.git.sourceforge.net/gitroot/aquosa/linux-rt-irmos
```

Alternatively, you can download one of the supported kernel releases from [`http://www.kernel.org`](<http://www.kernel.org>) (currently, we have a patch for the `2.6.30.10` series), and the corresponding IRMOS patch from the [AQuoSA web site](<http://sourceforge.net/projects/aquosa/files/>). Also, you need to properly configure the kernel, ensuring the following options are enabled (most of them are already enabled by default): 

```
RT_GROUP_SCHED
        GROUP_SCHED
        CGROUPS
        CGROUP_SCHED
        EXPERIMENTAL
        PREEMPT (recommended)
        CGROUP_CPUACCT (recommended)
        SCHED_DEBUG (recommended)
        HIGH_RES_TIMERS
        HZ_1000 (suggested)
```

If preferred, a few binary RPM/DEB kernel packages can also be conveniently downloaded from the [AQuoSA web site](<http://sourceforge.net/projects/aquosa/files/>). 

#### Usage

In order to use the realtime scheduler's capabilities, you need to mount the cgroup filesystem with something like: 

```
mkdir /cg
        mount -t cgroup -o cpu,cpuacct cgroup /cg
```

By default, up to 95% of the CPU power is allocated to standard POSIX realtime tasks in the root group, which doesn't leave much left over for reservations. So, before we can create a new group, we need to reduce the runtime for root-level tasks, e.g., lowering it to 200ms every 1s: 

```
echo 200000 > /cg/cpu.rt_rt_task_runtime_us
```

Now we can create a new group, with a reservation of 10ms every 100ms: 

```
mkdir /cg/g1
        echo 100000 > /cg/g1/cpu.rt_period_us
        echo 10000 > /cg/g1/cpu.rt_runtime_us
        echo 100000 > /cg/g1/cpu.rt_task_period_us
        echo 10000 > /cg/g1/cpu.rt_task_runtime_us
```

At this point, the new group has no associated tasks. We can attach a task by writing its Linux thread id (tid) to the `tasks` special file entry available in the group folder: 

```
echo 1421 > /cg/g1/tasks
```

Now the attached task has only been added to the group, but it still has its own scheduling class, defaulting to `SCHED_OTHER`. In order to exploit realtime scheduling, we need to assign to the task one of the realtime classes and a realtime priority: 

```
chrt -r -p 20 1421
```

At this point, the task is running with the configured scheduling guarantee (and limitation) of 10ms every 100ms. 

#### Usability and AQuoSA integration

As shown above, the interface towards the new realtime scheduling functionality is based on the cgroup filesystem. While constituting a perfectly usable interface for scripting languages and system administrators, this kind of interface makes programming realtime applications which exploit the new scheduler functionality quite cumbersome: in order to create new reservations, folders need to be created in the cgroup filesystem; for setting scheduling parameters, numbers need to be formatted and written to cgroup entries; for reading them, cgroup entries need to be read back and parsed; etc. The [libcgroup library](<http://libcg.sourceforge.net/>) may be of some help for such issues, but it carries non-negligible overhead into the applications. This may be especially troublesome for adaptive applications, e.g., multimedia ones, that might need to change dynamically the reservation parameters following the dynamic workload. 

Furthermore, when changing both scheduling parameters (runtime and period), operations need to be carried out in a proper order which depends on the previous values of the parameters themselves, otherwise the admission control logic may reject one of the intermediate steps. Also, while playing with the scheduling parameters (e.g., while tuning the application's performance), one is forced to use intermediate configurations which are highly undesirable. For example, reducing both the budget and the period by an order of magnitude, such as from `(100,200)` to `(10,20)`, one needs to reduce the runtime first, obtaining `(10,200)`, then the period. However, in the intermediate configuration the realtime task is likely to fail due to the insufficient resources being granted. 

Also, in the future, the number of parameters needed to configure the realtime scheduler's behavior is expected to (slightly) grow. What is needed from an application development perspective, is an atomic way of setting and changing them, possibly in the form of a user-space library (or system call) interface. 

However, the new scheduler is being integrated into the [AQuoSA open-source project](<http://aquosa.sourceforge.net>) (Adaptive Quality of Service Architecture), which makes a well-designed [user-space API](<http://aquosa.sourceforge.net/aquosa-docs/aquosa-qosres/html/group__QRES__LIB.html>) and adaptive reservations available to application developers as a dynamically linkable library. AQuoSA provides a user-space library which implements the the AQuoSA API on top of the cgroup-based operations needed to deal with the IRMOS scheduler, easing the task of coding applications that want to use it. More details on the AQuoSA integration can be found [here](<http://aquosa.sourceforge.net/news-2010-07-27.php>). 

#### Relationship with SCHED_DEADLINE

In addition to the IRMOS realtime scheduler, the Real-Time Systems Laboratory of Scuola Superiore Sant'Anna also collaborated with [Evidence](<http://www.evidence.eu.com>) in the implementation of another EDF-based realtime scheduler for Linux, [SCHED_DEADLINE](<http://lwn.net/Articles/356576/>). It is natural to wonder how these two schedulers differ: 

  * SCHED_DEADLINE allows for having one single task attached to an EDF reservation; this raises such issues as what to exactly do if the EDF task forks. Options range from setting the policy of the child to SCHED_OTHER, to setting it to SCHED_DEADLINE with the original bandwidth split in half between the father and child, to providing the child with an initial bandwidth (budget, or runtime) of zero, as happens by default in the latest implementation. 

The IRMOS scheduler has hierarchical scheduling capabilities, in that it allows for having a full-fledged POSIX realtime (sub-)scheduler nested inside each EDF-based reservation: when an EDF thread forks, the child keeps the same class (_FIFO or _RR) and priority as the parent and they both keep sharing the same EDF reservation. This allows for easy encapsulation of entire complex multi-threaded software components (e.g., an Apache web server, a KVM instance, etc.), but it also works with "traditional" realtime software components, such as a set of a few realtime threads with realtime priorities which constitute a single realtime component on the system (e.g., a control thread along with the IRQ threads related to its own I/O towards the controlled peripherals). If a strong assessment of the realtime schedulability of the system is needed, it is possible to use hierarchical realtime scheduling theory in order to analyze whether or not the individual RT threads will meet their deadlines. 

  * SCHED_DEADLINE uses a new system call interface, `sched_setscheduler_ex()`, which only allows for creating a reservation attached to a task. 

The IRMOS scheduler exploits the realtime throttling cgroup-based interface which was already there in the kernel, thus a new empty reservation is created by creating a folder in the cgroup filesystem, tasks are attached by adding their TIDs to the `tasks` file, the runtime and period are set by echoing their values into the corresponding file entries in the group folder, etc. 

  * SCHED_DEADLINE supports partitioned EDF, and we have a draft [global EDF implementation [PDF]](<http://etd.adm.unipi.it/theses/available/etd-05212010-224155/unrestricted/Lelli_Juri_Thesis.pdf>) made as part of the Master Degree Thesis in Computer Engineering of Juri Lelli. 

In the IRMOS scheduler, reservations apply to all CPUs, and it supports Global Fixed Priority tasks nested inside partitioned EDF-based reservations. If a RT task exhausts the budget (runtime) of the EDF scheduler over a CPU, it can still run exploiting the budget available in the same reservation over another CPU (migrating the task or the bandwidth are both available options). However, affinity masks can be used in order to better control over which CPUs realtime tasks will be able to migrate. 

  * SCHED_DEADLINE is implemented as a completely new scheduling class, while the IRMOS scheduler is a modification to the already available POSIX realtime scheduling classes. 

  * SCHED_DEADLINE also supports soft (work-conserving) resource reservations, while the IRMOS scheduler does not (however, it is planned as future work). 

Even if the two projects are currently completely separated, there is a good basis for having a common EDF-based realtime scheduling infrastructure, that might be used by using different user-space APIs in the two cases. 

#### Conclusions

In the future, we plan to improve the scheduler on various sides. Concerning the user-space API, the cgroup-based interaction model already proved to suffer from major limitations. For example, the absence of an atomic way to set at the same time multiple scheduling parameters constitutes a major limitation of the current interface. Also, we plan to develop more options in the realtime scheduling model, such as: 

  * Adding soft resource reservations, allowing for work-conserving reservations coexisting with non-work-conserving ones. 

  * Improving the access-control model, making the realtime scheduling capabilities more easily accessible to unprivileged applications. 

  * Adding the possibility to specify a desired budget (run time), in addition to the minimum guaranteed one subject to admission control, which could be used for implementing adaptive reservations, which, in turn, could be useful for applications showing significant workload fluctuations at run time. 

  * Adding some form of deadline inheritance for better addressing the well-known priority inheritance problem, e.g., by means of the [Multi-processor BandWidth Inheritance](<http://retis.sssup.it/~tommaso/eng/publications#L_ECRTS10>) (MBWI) protocol, or some variation on the concept. 

All of the above improvements would go into the direction of enhancing usability of the realtime scheduler by common multimedia applications. These would be the applications taking most of the benefits from the exploitation of realtime scheduling capabilities of the Linux kernel, such as the ones described above.
