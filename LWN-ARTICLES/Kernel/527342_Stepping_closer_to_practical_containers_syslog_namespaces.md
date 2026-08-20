---
title: "Stepping closer to practical containers: \"syslog\" namespaces"
url: https://lwn.net/Articles/527342/
date: "December 5, 2012"
category: "Namespaces-Syslog namespaces; printk"
author: "By Michael Kerrisk December 5, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Michael Kerrisk**  
December 5, 2012

The abstract goal of [containers](<https://lwn.net/Articles/524952/>) is, in effect, to provide a group of processes with the illusion that that they are the only processes on the system. When fully implemented, this feature has the potential to realize many practical benefits, such as light-weight virtualization and [checkpoint/restore](<https://lwn.net/Articles/524952/>). 

In order to give the processes in a container the illusion that there are no other processes on the system, various global system resources must be wrapped in abstractions that make it appear that each container has its own instance of the resources. This has been achieved by the addition of "namespaces" for a number of global resources. Each namespace provides an isolated view of a particular global resource to the set of processes that are members of that namespace. 

Step by step, more and more global resources have been wrapped in namespaces, and before we look at another step in this path it's worth reviewing the progress to date. 

#### Namespaces so far

The [first step](<http://lwn.net/2001/0301/a/namespaces.php3>) in the journey was mount namespaces, which can be used to provide a group of processes with a private view of the mount points that make up the filesystem hierarchy. Mount namespaces first appeared in the mainline kernel in 2002, with the release of Linux 2.4.19. The `clone()` flag used to create mount namespaces was given the rather generic name `CLONE_NEWNS` for "new namespace", implying that no one was then really considering the possibility that there might be other kinds of namespaces; at that time, of course, containers were no more than a gleam in the eyes of some developers. 

However, as the concept of containers took hold, a number of other namespaces have followed. [Network namespaces](<https://lwn.net/Articles/219794/>) were added to provide a group of processes with a private view of the network (network devices, IP addresses, IP routing tables, port number space, and so on). [PID namespaces](<https://lwn.net/Articles/259217/>) isolated the global "PID number space" resource, so that processes in separate PID namespaces can have the same PIDs--in particular, each namespace can have its own 'init' (PID 1), the "ancestor of all processes". PID namespaces also allow techniques such as freezing the processes in a container and then restoring them on another system while maintaining the same PIDs. 

Several other global resources have likewise been wrapped in namespaces, so that there are also [IPC namespaces](<https://lwn.net/Articles/187274/>) (initially implemented to isolate System V IPC identifiers and [later](<https://lwn.net/Articles/312232/>) to isolate instances of the virtual filesystems used in the implementation of POSIX message queues) and [UTS namespaces](<https://lwn.net/Articles/179345/>) (which wrap the `nodename` and `domainname` identifiers returned by [`uname(2)`](<http://man7.org/linux/man-pages/man2/uname.2.html>)). Work on one of the more complex namespaces, [user namespaces](<https://lwn.net/Articles/491310/>), was started in about Linux 2.6.23 and seems to be [edging](<https://lwn.net/Articles/525709/>) towards completion. When complete, user namespaces will allow per-namespace mappings of user and group IDs, so that, for example, it will be possible for a process to be root inside a container without having root privileges in the system as a whole. 

Of course, a Linux system has a large number of global resources, each of which could conceivably be wrapped in a namespace. At the more extreme end, for example, even a resource such as the system time could be wrapped, so that different containers could maintain different concepts of the time. (A time namespace was [once proposed](<https://lwn.net/Articles/180375/>), but the implementation was not merged.) The trick is to determine the minimum set of resources that need to be wrapped for the _practical_ implementation of containers. (Of course, this "minimum set" may well grow over time, as people develop new uses for containers.) A related question is how those wrappings should be grouped so as to avoid an explosion of namespaces that would increase application complexity. So, for example, System V IPC and POSIX message queues could conceivably have been wrapped in different namespaces, but the kernel developers concluded that it makes practical sense to group them in a single "IPC" namespace. 

#### The global kernel log problem

What is necessary for the practical implementation of containers sometimes only becomes clear when one starts doing, well, practical things. Thus, it was that in early 2010 Jean-Marc Pigeon [reported](<https://lwn.net/Articles/527372/>) that he had written a small utility to build containers using the `clone()` system call that worked fine, except that ""HOST and all containers share the SAME /proc/kmsg, meaning kernel syslog information are scrambled (useless)"". 

What Jean-Marc was discovering is that the kernel log is one of the global resources that is not wrapped in a namespace. He went on to note another ill-effect: ""I have in iptables, reject packet logging on the HOST, [but as soon as] rsyslog is started on one container, I can't see my reject packet log any more."" In other words, starting a `syslog` daemon on the host or any container sucks up all of the kernel log messages produced on the host or in any container. The point here about `iptables` is particularly relevant: the inability to isolate kernel log messages from `iptables` is a significant practical problem when trying to employ the network namespaces facility that the kernel already provides. 

In response to Jean-Marc's question about how the problem could be fixed, Serge Hallyn [replied](<https://lwn.net/Articles/527383/>): 

Well, the results of do_syslog() should be containerized. Kernel messages (oopses for instance) should always go to the initial container. Shouldn't be hard to do, but the question is what do we tie it to? User namespace? Network namespace? … I'm tempted to say userns makes the most sense - if you start a new userns you likely always want private syslog, whereas with netns and pidns you may not. 

`do_syslog()` is the kernel function that encapsulates the main logic of the [`syslog(2)`](<http://man7.org/linux/man-pages/man2/syslog.2.html>) system call. That system call retrieves messages from the kernel log ring buffer (and performs a range of control operations on the log buffer) that is populated by messages created using the kernel's `printk()` function. Thus, though discussions on this topic have tended to use the term "syslog namespace", that is something of a misnomer: what is really meant is wrapping the kernel log resource in a namespace. 

To avoid possible confusion, it is probably worth noting that the `syslog(2)` system call is a quite different thing from the [`syslog(3)`](<http://man7.org/linux/man-pages/man3/syslog.3.html>) library function, which writes messages to the UNIX domain datagram socket (`/dev/log`) from which the user-space syslog daemon (`rsyslogd` or similar) retrieves messages. (Because of this collision of names, the GNU C library exposes the `syslog(2)` system call under a quite different name: `klogctl()`.) A picture helps clarify things: 

> ![\[Interrelationship of
logging primitives\]](https://static.lwn.net/images/2012/syslog.png)

#### First attempts at a solution

In the event, "containerizing" `do_syslog()` turned out to be more difficult than Serge thought. His [first shot](<https://lwn.net/Articles/527386/>) at addressing the problem (a "gross hack" to ""provide each user namespace with its own syslog ring buffer"") quickly uncovered [a further difficulty](<https://lwn.net/Articles/527388/>): the kernel's `printk()` is sometimes called in contexts where there is no way to determine in which of the per-namespace ring buffers a message should be logged. For example, if the kernel is executing a network interrupt (to process an incoming network packet) and wants to log a message, that message should _not_ be sent to the per-namespace kernel log of the interrupted process. Rather, the message should be sent to the kernel log associated with the network namespace for the network device; however, the kernel data structures provide no way to obtain a reference to that kernel log. 

Jean-Marc himself also made [an attempt](<https://lwn.net/Articles/527406/>) at implementing a solution. However, Serge [pointed out](<https://lwn.net/Articles/527409/>) that Jean-Marc's patch suffered some of the same problems as his own earlier attempt. Serge went on to describe what he thought would be the correct solution, which would require the creation of a separate syslog namespace. His proposed solution can be paraphrased as follows: 

  1. The core of `vprintk_emit()` (which contains most of implementation of the `printk()` function) should be moved into a new `nsvprintk_emit()` function that takes an argument that specifies a syslog namespace. 

  2. `vprintk_emit()` would then become a wrapper around `nsvprintk_emit()` that specifies the "initial" syslog namespace (i.e., the syslog namespace of the host system). 

  3. A namespace-aware version of `printk()`, called (say) `nsprintk()`, should be created. That function would take a syslog namespace argument and pass it to `nsvprintk_emit()`. 

  4. The kernel log ring buffer should be "containerized" as per Serge's initial patch. Thus each syslog namespace would have its own ring buffer, and `syslog(2)` would operate on the per-namespace ring buffer of the calling process. 

  5. At call sites in the kernel code where it is not appropriate to use the syslog namespace of the currently executing process, calls to `printk()` should be replaced with calls to `nsprintk()` that pass a suitable syslog namespace argument. 

Although Jean-Marc made a few more efforts to rework his patch in the following weeks, the effort ultimately petered out without much further comment or consensus on a solution. It seems that Serge and [other kernel developers](<https://lwn.net/Articles/527411/>) realized that the problem was more complex than first thought and they had neither the time to implement a solution themselves nor to help Jean-Marc toward implementing a solution. 

The main difficulty lies in the last of the points above, and its solution was not really elaborated in Serge's mail. The kernel data structures and code need to be modified to add suitable hooks to handle the "no current process context problem"--the cases where `printk()` is called from a context in which the currently executing process can't be used to identify a suitable syslog namespace to which a message should be logged. 

#### Restarting work on a solution

Work in this area then seems to have gone quiet for more than two years, until a few days ago when Serge proposed [a new proof-of-concept patch set](<https://lwn.net/Articles/525629/>), pretty much along the lines he described two years earlier. His description of the patch noted that: 

The syslog ns is tied to a user namespace. You must create a new user namespace before you can create a new sylog ns. The syslog ns is created through a new command (11) to the __NR_syslog system call. 

Once a task enters a new syslog ns, it's "dmesg", "dmesg -c" and /dev/kmsg actions affect only itself, so that user-created syslog messages no longer are confusingly combined in the host's syslog. 

In other words, Serge's patch provides isolation for the kernel log by implementing a new dedicated namespace for that purpose (rather than providing the isolation by attaching the implementation to one of the existing namespaces). Each syslog namespace instance would be tied to a particular user namespace. 

Normally, new namespaces of each type are created by suitable flags to the `clone()` system call. Thus, for example, there are clone flags such as `CLONE_NEWUTS` and `CLONE_NEWUSER`. However, a while ago, the kernel developers realized that the flag space for `clone()` was exhausted. (Providing additional flag space was one of the motivations behind the [proposal](<https://lwn.net/Articles/362371/>) to add an `eclone()` system call, a proposal that was ultimately unsuccessful.) For this reason, Serge proposed instead to use a new command to the `syslog()` system call to create syslog namespace instances. 

Serge went on to note: 

"printk" itself always goes to the initial syslog_ns, and consoles belong only to the initial syslog_ns. However printks relating to a specific network namespace, for instance, can now be targeted to the syslog ns for the user ns which owns the network ns, aiding in debugging in a container. 

Serge's patch would solve the "no current process context problem" as follows. As noted above, this case is handled by an `nsprintk()`-style function that takes an argument (of type `struct syslog_ns *)` that identifies the syslog namespace to which the log message should be sent. The value for that argument can be obtained via the `struct net` structure for the network namespace instance: in the current user namespace implementation ([git tree](<http://git.kernel.org/?p=linux/kernel/git/ebiederm/user-namespace.git>)), when a network namespace is created using `clone()`, a pointer to the corresponding user namespace instance of the caller is stored in the `net` structure. Serge's patch in turn provides a linkage from that user namespace structure to the corresponding syslog namespace. 

Eric Biederman, the maintainer of the user namespace git tree, [agreed](<https://lwn.net/Articles/527669/>) with Serge's overall approach, but queried one particular point: 

I am not a fan of how this ties into the user namespace. I would prefer closer or looser ties. The recursive reference count loop where a userns refers to a syslogns and that syslogns refers to the same userns is unpleasant. 

In Serge's implementation, the syslog and user namespaces are maintained as separate structures, but, as the recursive pointers between the two namespace structures and the need to create a new user namespace before creating a syslog namespace indicate, instances of each namespace are not truly independent. In Eric's view then, the syslog and user namespace structures should either be more fully decoupled, or they should be much more tightly coupled. 

Eric went on later to [note](<https://lwn.net/Articles/527671/>) that: 

There is an argument to be made that syslog messages are the kind of security identifiers like uid, gids, and keys that should be part of a user namespace. I'm not fully convinced but there are some DOS attacks that would naturally prevent. 

The discussion ultimately led Serge to [conclude](<https://lwn.net/Articles/527673/>) that the syslog resource should instead be grouped as part of the user namespace rather than as a separate namespace: 

I can't really think of a good case for not putting the syslogns straight into the userns (i.e. not having a separate syslogns), so I'd say let's go that route. 

Serge's patch seems to have inspired another group to try implementing syslog namespaces. A couple of days after Serge's patch, Rui Xiang [posted](<https://lwn.net/Articles/525728/>) some patches that he and his colleague Libo Chen had developed to implement similar functionality. Rui began by noting a couple of the obvious differences in their patch set: 

In Serge's patch [...] syslog_namespace was tied to a user namespace. We add syslog_ns tied to nsproxy instead, and implement ns_printk in ip_table context. 

We add syslog_namespace as a part of nsproxy, and a new flag CLONE_SYSLOG to unshare syslog area. 

Using `nsproxy` is the conventional way of dealing with the namespaces associated with a process: it is a structure that contains pointers to structures describing each of the namespaces that a process is associated with. This contrasts with Serge's original approach, which hung the syslog namespace off the user namespace. 

Rui's team also took advantage of a detail that Serge perhaps overlooked: there happens to be one spare bit in the flag space for `clone()` because the `CLONE_STOPPED` flag was removed several kernel releases ago. Therefore, Rui's team repurposed that bit. Normally, it would not be safe to recycle flag bits in this way, but the `CLONE_STOPPED` flag has a special history. It was initially proposed for use specifically in the NPTL threading implementation, but the final implementation abandoned the flag in favor of a different approach. As such, `CLONE_STOPPED` is likely never to have had serious user-space users. 

Unsurprisingly, the overall approaches of the two patch sets have many similarities, but there are differences in details such as how a syslog namespace is associated with a `struct net` in order to solve the "no current process context problem". 

Although kernel flame wars between competing implementations are what often make the biggest headlines in the online press, the subsequent exchange between Serge, Rui, and Libo demonstrated that life on developer mailing lists is usually more cordial. Serge [asked](<https://lwn.net/Articles/527717/>): 

I understand that user namespaces aren't 100% usable yet, but looking long term, is there a reason to have the syslog namespace separate from user namespace? 

In response, [Rui](<https://lwn.net/Articles/527718/>) noted: 

Actually we don't have strong preference. We'll think more about it. Hope we can make consensus with Eric. 

That in turn led Serge to [ask](<https://lwn.net/Articles/527722/>) Rui and Libo if his patch set might suffice for their needs, with the gracious note that: 

I'm not at all wedded to my patchset. I'm happy to go with something else entirely. My set was just a proof of concept. 

There is one other notable difference in functionality between the two patch sets. In Serge's patch set, system consoles belonged (by intention) only to the _initial_ syslog namespace, meaning that kernel log messages from other syslog namespace instances can't be displayed on consoles. By contrast, Rui and Libo's patches include consoles in the syslog namespace, so that kernel messages from syslog namespaces other than the initial namespace can be displayed on consoles. Rui and Libo would like this functionality in order to be able to obtain kernel log messages from containers when monitoring embedded devices that provide access to the console over a serial port. 

The summary of the discussion is that there are useful pieces in both patches. Serge [plans](<https://lwn.net/Articles/527729/>) to revise his patch to merge the syslog namespace functionality into user namespaces, add the console functionality desired by Rui and Libo, and add some in-kernel uses of the namespace-aware `printk()` interface as a proof-of-concept for the implementation (as was done in the patches by Rui and Libo). 

#### Concluding remarks

The history of the work to provide syslog namespaces (or as it might better be termed, namespace isolation for the kernel log) presents a microcosm of work on namespaces in general. As has often been the case, the implementation of namespaces often turns out to be surprisingly complex. Much of that complexity hinges on detailed questions of functionality (for example, the behavior of consoles in this case) and the question of whether resources should be grouped inside a new namespace or within an existing namespace. In the case of syslog namespaces, it looks like a number of decisions have been made; there will probably be a few more rounds of patches, but there seems to be general consensus on the direction forward. Thus, there is a reasonable chance that proper namespace isolation of kernel logging will appear in the kernel sometime around Linux 3.9 or soon afterward.
