---
title: Process containers
url: https://lwn.net/Articles/236038/
date: "May 29, 2007"
category: "Containers; Virtualization-Containers"
author: 
---

Back in September, LWN took a look at [Rohit Seth's containers patch](<http://lwn.net/Articles/199643/>). Since that time, containers development has moved on to Paul Menage who, like Rohit, posts from a google.com address. The patch has evolved considerably, to the point that Rohit's name no longer appears within it. As of the recently posted [containers V10 patch](<http://lwn.net/Articles/236032/>), this mechanism is reaching a reasonably mature state. 

This patch introduces a couple of new concepts into the kernel. The first one has an old name: "subsystem". Fortunately, the driver core has just removed its "subsystem" concept, leaving the term free. In the container patch, a subsystem is some part of the kernel which might have an interest in what groups of processes are doing. Chances are that most subsystems will be involved with resource management; for example, the container patch turns the Linux cpusets mechanism (which binds processes to specific groups of processors) into a subsystem. 

A "container" is a group of processes which shares a set of parameters used by one or more subsystems. In the cpuset example, a container would have a set of processors which it is entitled to use; all processes within the container inherit that same set. Other (not yet existing) subsystems could use containers to enforce limits on CPU time, I/O bandwidth usage, memory usage, filesystem visibility, and so on. Containers are hierarchical, in that one container can hold others. 

![\[container hierarchy\]](https://static.lwn.net/images/ns/kernel/ContainerHierarchy.png) As an example, consider the simple hierarchy to the right. A server used to host containerized guests could establish two top-level containers to control the usage of CPU time. Guests, perhaps, could be allowed 90% of the CPU, but the administrator may want to place system tasks in a separate container which will always get at least 10% of the processor - that way, the mail will continue to be delivered regardless of what the guests are doing. Within the "Guests" container, each individual guest has its own container with specific CPU usage policies. 

The container mechanism is not limited to a single hierarchy; instead, the administrator can create as many hierarchies as desired. So, for example, the administrator of the system described above could create an entirely different hierarchy for the control of network bandwidth usage. By default, all processes would be in the same container, but it is possible to set up policy which would shift processes to a different container when they run a specific application. So a web browser might be moved into a container which gets a relatively high portion of the available bandwidth while Bittorrent clients find themselves relegated to an unhappy container with almost no bandwidth available. 

Different container hierarchies need not resemble each other in any way. Each hierarchy has one or more subsystems associated with it; a subsystem can only be attached to a single hierarchy. If there is more than one hierarchy, each process in the system will be in more than one container - one in each hierarchy. 

The administration of containers is performed through a special virtual filesystem. The documentation suggests that it could be mounted on `/dev/container`, which is a bit strange; it has nothing to do with devices. One container filesystem instance will be mounted for each hierarchy to be created. The association of subsystems with hierarchies is done at mount time, by way of mount options. By default, all known subsystems are associated with a hierarchy, so a command like: 

```
mount -t container none /containers
```

would create a single container hierarchy with all known subsystems on `/containers`. A setup like the one described above, instead, could be created with something like: 

```
mount -t container -o cpu cpu /containers/cpu
        mount -t container -o net net /containers/net
```

The desired subsystems for each container hierarchy are simply provided as options at mount time. Note that the "cpu" and "net" subsystems mentioned above do not actually exist in the current container patch set. 

Creating new containers is just a matter of making a directory in the appropriate spot in the hierarchy. Containers have a file called `tasks`; reading that file will yield a list of all processes currently in the container. A process can be added to a container by writing its ID to the `tasks` file. So a simple way to create a container and move a shell into it would be: 

```
mkdir /containers/new_container
        echo $$ > /containers/new_container/tasks
```

Subsystems can add files to containers for use in setting resource limits or otherwise controlling how the subsystem works. For example, the cpuset subsystem (which does exist) adds a file called `cpus` containing the list of CPUs established for that container; there are several other files added as well. 

It's worth noting that the container patch does not add a single system call; all of the management is performed through the virtual filesystem. 

With a basic container mechanism in place, most of the action in the future is likely to be in the creation of new subsystems. One can imagine, for example, hooking the existing process ID virtualization code into containers, as well as adding no end of resource controllers. The creation of a subsystem is relatively straightforward; the subsystem code starts by creating and registering a `container_subsys` structure. That structure contains an integer `subsys_id` field which should be set to the subsystem's specific ID number; these numbers are set staticly in `<linux/container_subsys.h>`. Implicit in this arrangement is that subsystems must be built into the kernel; there is no provision for adding subsystems as loadable modules. 

Each subsystem defines a set of methods to be used by the container code, beginning with: 

```
int (*create)(struct container_subsys *ss, struct container *cont);
        int (*populate)(struct container_subsys *ss, struct container *cont);
        void (*destroy)(struct container_subsys *ss, struct container *cont);
```

These three are called whenever a container is created or destroyed; this is the chance for the subsystem to set up any bookkeeping it will need for the new container (or clean up for a container which is going away). The `populate()` method is called after the successful creation of a new container; its purpose is to allow the subsystem to add management files to that container. 

Four methods are for the addition and removal of processes: 

```
int (*can_attach)(struct container_subsys *ss, struct container *cont, 
                          struct task_struct *tsk);
        void (*attach)(struct container_subsys *ss, struct container *cont,
    		   struct container *old_cont, struct task_struct *tsk);
        void (*fork)(struct container_subsys *ss, struct task_struct *task);
        void (*exit)(struct container_subsys *ss, struct task_struct *task);
```

If a process is explicitly added to a container after creation, the container code will call `can_attach()` to determine whether the addition should succeed. If the subsystem allows the action to happen, it should have performed any needed allocations to ensure that the subsequent `attach()` call succeeds. When a process forks, `fork()` will be called to add the new child to the container. Exiting processes call `exit()` to allow the subsystem to clean up. 

Clearly, there's more to the interface than described here; see [the thorough documentation file](<https://lwn.net/Articles/236078/>) packaged with the patch for much more detail. Your editor would not venture a guess as to when this code might be merged, but it does seem that this is the mechanism that the containers community has decided to push. So, sooner or later, it will likely be contained within the mainline.
