---
title: "Namespaces in operation, part 3: PID namespaces"
url: https://lwn.net/Articles/531419/
date: "January 16, 2013"
category: "Namespaces-PID namespaces"
author: "By Michael Kerrisk January 16, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Michael Kerrisk**  
January 16, 2013

Following on from our two earlier namespaces articles ([Part 1: namespaces overview](<https://lwn.net/Articles/531114/>) and [Part 2: the namespaces API](<https://lwn.net/Articles/531381/>)), we now turn to look at PID namespaces. The global resource isolated by PID namespaces is the process ID number space. This means that processes in different PID namespaces can have the same process ID. PID namespaces are used to implement containers that can be migrated between host systems while keeping the same process IDs for the processes inside the container. 

As with processes on a traditional Linux (or UNIX) system, the process IDs _within_ a PID namespace are unique, and are assigned sequentially starting with PID 1. Likewise, as on a traditional Linux system, PID 1--the `init` process--is special: it is the first process created within the namespace, and it performs certain management tasks within the namespace. 

#### First investigations

A new PID namespace is created by calling [`clone()`](<http://man7.org/linux/man-pages/man2/clone.2.html>) with the `CLONE_NEWPID` flag. We'll show a simple example program that creates a new PID namespace using `clone()` and use that program to map out a few of the basic concepts of PID namespaces. The complete source of the program (`pidns_init_sleep.c`) can be found [here](<https://lwn.net/Articles/532741/>). As with the previous article in this series, in the interests of brevity, we omit the error-checking code that is present in the full versions of the example program when discussing it in the body of the article. 

The main program creates a new PID namespace using `clone()`, and displays the PID of the resulting child: 

```
child_pid = clone(childFunc,
                        child_stack + STACK_SIZE,   /* Points to start of
                                                       downwardly growing stack */
                        CLONE_NEWPID | SIGCHLD, argv[1]);
    
        printf("PID returned by clone(): %ld\n", (long) child_pid);
```

The new child process starts execution in `childFunc()`, which receives the last argument of the `clone()` call (`argv[1]`) as its argument. The purpose of this argument will become clear later. 

The `childFunc()` function displays the process ID and parent process ID of the child created by `clone()` and concludes by executing the standard `sleep` program: 

```
printf("childFunc(): PID = %ld\n", (long) getpid());
        printf("ChildFunc(): PPID = %ld\n", (long) getppid()); 
        ...
        execlp("sleep", "sleep", "1000", (char *) NULL);
```

The main virtue of executing the `sleep` program is that it provides us with an easy way of distinguishing the child process from the parent in process listings. 

When we run this program, the first lines of output are as follows: 

```
$ **su**         # Need privilege to create a PID namespace
        Password: 
        # **./pidns_init_sleep /proc2**
        PID returned by clone(): 27656
        childFunc(): PID  = 1
        childFunc(): PPID = 0
        Mounting procfs at /proc2
```

The first two lines line of output from `pidns_init_sleep` show the PID of the child process from the perspective of two different PID namespaces: the namespace of the caller of `clone()` and the namespace in which the child resides. In other words, the child process has two PIDs: 27656 in the parent namespace, and 1 in the new PID namespace created by the `clone()` call. 

The next line of output shows the parent process ID of the child, within the context of the PID namespace in which the child resides (i.e., the value returned by `getppid()`). The parent PID is 0, demonstrating a small quirk in the operation of PID namespaces. As we detail below, PID namespaces form a hierarchy: a process can "see" only those processes contained in its own PID namespace and in the child namespaces nested below that PID namespace. Because the parent of the child created by `clone()` is in a different namespace, the child cannot "see" the parent; therefore, `getppid()` reports the parent PID as being zero. 

For an explanation of the last line of output from `pidns_init_sleep`, we need to return to a piece of code that we skipped when discussing the implementation of the `childFunc()` function. 

#### `/proc/`_PID_ and PID namespaces

Each process on a Linux system has a `/proc/`_PID_ directory that contains pseudo-files describing the process. This scheme translates directly into the PID namespaces model. Within a PID namespace, the `/proc/`_PID_ directories show information only about processes within that PID namespace or one of its descendant namespaces. 

However, in order to make the `/proc/`_PID_ directories that correspond to a PID namespace visible, the proc filesystem ("procfs" for short) needs to be mounted from within that PID namespace. From a shell running inside the PID namespace (perhaps invoked via the `system()` library function), we can do this using a `mount` command of the following form: 

```
# **mount -t proc proc /mount_point**
```

Alternatively, a procfs can be mounted using the `mount()` system call, as is done inside our program's `childFunc()` function: 

```
mkdir(mount_point, 0555);       /* Create directory for mount point */
        mount("proc", mount_point, "proc", 0, NULL);
        printf("Mounting procfs at %s\n", mount_point);
```

The `mount_point` variable is initialized from the string supplied as the command-line argument when invoking `pidns_init_sleep`. 

In our example shell session running `pidns_init_sleep` above, we mounted the new procfs at `/proc2`. In real world usage, the procfs would (if it is required) usually be mounted at the usual location, `/proc`, using either of the techniques that we describe in a moment. However, mounting the procfs at `/proc2` during our demonstration provides an easy way to avoid creating problems for the rest of the processes on the system: since those processes are in the same _mount_ namespace as our test program, changing the filesystem mounted at `/proc` would confuse the rest of the system by making the `/proc/`_PID_ directories for the root PID namespace invisible. 

Thus, in our shell session the procfs mounted at `/proc` will show the _PID_ subdirectories for the processes visible from the parent PID namespace, while the procfs mounted at `/proc2` will show the _PID_ subdirectories for processes that reside in the child PID namespace. In passing, it's worth mentioning that although the processes in the child PID namespace will be able to see the _PID_ directories exposed by the `/proc` mount point, those PIDs will not be meaningful for the processes in the child PID namespace, since system calls made by those processes interpret PIDs in the context of the PID namespace in which they reside. 

Having a procfs mounted at the traditional `/proc` mount point is necessary if we want various tools such as `ps` to work correctly inside the child PID namespace, because those tools rely on information found at `/proc`. There are two ways to achieve this without affecting the `/proc` mount point used by parent PID namespace. First, if the child process is created using the `CLONE_NEWNS` flag, then the child will be in a different mount namespace from the rest of the system. In this case, mounting the new procfs at `/proc` would not cause any problems. Alternatively, instead of employing the `CLONE_NEWNS` flag, the child could change its root directory with `chroot()` and mount a procfs at `/proc`. 

Let's return to the shell session running `pidns_init_sleep`. We stop the program and use `ps` to examine some details of the parent and child processes within the context of the parent namespace: 

```
^Z                          _Stop the program, placing in background_
        [1]+  Stopped                 ./pidns_init_sleep /proc2
        # **ps -C sleep -C pidns_init_sleep -o "pid ppid stat cmd"**
          PID  PPID STAT CMD
        27655 27090 T    ./pidns_init_sleep /proc2
        27656 27655 S    sleep 600
```

The "PPID" value (27655) in the last line of output above shows that the parent of the process executing `sleep` is the process executing `pidns_init_sleep`. 

By using the `readlink` command to display the (differing) contents of the `/proc/`_PID_` /ns/pid` symbolic links ([explained](<https://lwn.net/Articles/531381/#proc_pid_ns>) in last week's article), we can see that the two processes are in separate PID namespaces: 

```
# **readlink /proc/27655/ns/pid**
        pid:[4026531836]
        # **readlink /proc/27656/ns/pid**
        pid:[4026532412]
```

At this point, we can also use our newly mounted procfs to obtain information about processes in the new PID namespace, from the perspective of that namespace. To begin with, we can obtain a list of PIDs in the namespace using the following command: 

```
# **ls -d /proc2/[1-9]***
        /proc2/1
```

As can be seen, the PID namespace contains just one process, whose PID (inside the namespace) is 1. We can also use the `/proc/`_PID_` /status` file as a different method of obtaining some of the same information about that process that we already saw earlier in the shell session: 

```
# **cat /proc2/1/status | egrep '^(Name|PP*id)'**
        Name:   sleep
        Pid:    1
        PPid:   0
```

The `PPid` field in the file is 0, matching the fact that `getppid()` reports that the parent process ID for the child is 0. 

#### Nested PID namespaces

As noted earlier, PID namespaces are hierarchically nested in parent-child relationships. Within a PID namespace, it is possible to see all other processes in the same namespace, as well as all processes that are members of descendant namespaces. Here, "see" means being able to make system calls that operate on specific PIDs (e.g., using `kill()` to send a signal to process). Processes in a child PID namespace cannot see processes that exist (only) in the parent PID namespace (or further removed ancestor namespaces). 

A process will have one PID in each of the layers of the PID namespace hierarchy starting from the PID namespace in which it resides through to the root PID namespace. Calls to `getpid()` always report the PID associated with the namespace in which the process resides. 

We can use the program shown [here](<https://lwn.net/Articles/532745/>) (`multi_pidns.c`) to show that a process has different PIDs in each of the namespaces in which it is visible. In the interests of brevity, we will simply explain what the program does, rather than walking though its code. 

The program recursively creates a series of child process in nested PID namespaces. The command-line argument specified when invoking the program determines how many children and PID namespaces to create: 

```
# **./multi_pidns 5**
```

In addition to creating a new child process, each recursive step mounts a procfs filesystem at a uniquely named mount point. At the end of the recursion, the last child executes the `sleep` program. The above command line yields the following output: 

```
Mounting procfs at /proc4
        Mounting procfs at /proc3
        Mounting procfs at /proc2
        Mounting procfs at /proc1
        Mounting procfs at /proc0
        Final child sleeping
```

Looking at the PIDs in each procfs, we see that each successive procfs "level" contains fewer PIDs, reflecting the fact that each PID namespace shows only the processes that are members of that PID namespace or its descendant namespaces: 

```
^Z                           _Stop the program, placing in background_
        [1]+  Stopped            ./multi_pidns 5
        # **ls -d /proc4/[1-9]***        _Topmost PID namespace created by program_
        /proc4/1  /proc4/2  /proc4/3  /proc4/4  /proc4/5
        # **ls -d /proc3/[1-9]***
        /proc3/1  /proc3/2  /proc3/3  /proc3/4
        # **ls -d /proc2/[1-9]***
        /proc2/1  /proc2/2  /proc2/3
        # **ls -d /proc1/[1-9]***
        /proc1/1  /proc1/2
        # **ls -d /proc0/[1-9]***        _Bottommost PID namespace_
        /proc0/1
```

A suitable `grep` command allows us to see the PID of the process at the tail end of the recursion (i.e., the process executing `sleep` in the most deeply nested namespace) in all of the namespaces where it is visible: 

```
# **grep -H 'Name:.*sleep' /proc?/[1-9]*/status**
        /proc0/1/status:Name:       sleep
        /proc1/2/status:Name:       sleep
        /proc2/3/status:Name:       sleep
        /proc3/4/status:Name:       sleep
        /proc4/5/status:Name:       sleep
```

In other words, in the most deeply nested PID namespace (`/proc0`), the process executing `sleep` has the PID 1, and in the topmost PID namespace created (`/proc4`), that process has the PID 5. 

If you run the test programs shown in this article, it's worth mentioning that they will leave behind mount points and mount directories. After terminating the programs, shell commands such as the following should suffice to clean things up: 

```
# **umount /proc?**
        # **rmdir /proc?**
```

#### Concluding remarks

In this article, we've looked in quite some detail at the operation of PID namespaces. In the next article, we'll fill out the description with a discussion of the PID namespace `init` process, as well as a few other details of the PID namespaces API.
