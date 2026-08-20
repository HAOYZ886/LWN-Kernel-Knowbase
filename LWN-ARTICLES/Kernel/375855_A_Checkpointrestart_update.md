---
title: A Checkpoint/restart update
url: https://lwn.net/Articles/375855/
date: "February 24, 2010"
category: Checkpointing; Containers
author: "By Jonathan Corbet February 24, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 24, 2010

It has been exactly one year since LWN last [checked up on the checkpoint/restart patch set](<http://lwn.net/Articles/320508/>). This code has just been [reposted](<http://lwn.net/Articles/375716/>) with a request for inclusion into the -mm tree, so it seems like an opportune time to restart our coverage of it. A lot of progress has been made on this front over the last year, but checkpoint/restart remains a difficult task which can probably never be implemented completely. 

"Checkpointing" refers to the act of saving the state of a group of processes to a file, with the intent of restarting those processes at some future time. For many years, checkpointing has been used to save the state of long-running calculations to avoid losing work should the system fail. More recently, it has become a desired part of the virtualization toolkit, enabling the live migration of processes between physical hosts. The checkpoint/restart developers also see other potential advantages, such as the ability to quickly launch a set of processes on demand from a checkpoint image. 

This patch set addresses checkpoint/restart in the containers context. In the context of full virtualization, checkpointing is relatively easy; the system just needs to save the entire memory image associated with the virtual machine and a bit of associated data. The "containers" model of virtualization tends to be messier in almost every way, and checkpointing is no exception. There is no memory image to be saved in one big chunk; instead, the kernel must track down every bit of state associated with the checkpointed processes and save it independently. When it works, it can be faster and more efficient than full virtual machine checkpointing; the checkpoint image will be much smaller. But getting it to work is a challenge. The complexity of this task can be seen in [the current checkpoint/restart tree](<http://www.linux-cr.org/git/?p=linux-cr.git;a=summary>), which, despite being far from a complete solution of the problem, is a 27,000-line diff from 2.6.33-rc8. 

#### Checkpointing

To checkpoint a group of processes, the following new system call is used: 

```
int checkpoint(pid_t pid, int fd, unsigned long flags, int logfd);
```

The `pid` parameter identifies the top-level process to be checkpointed; all children of that process will also be included in the checkpoint image, which will be written to the file indicated by `fd`. There is currently only one possible flag value, `CHECKPOINT_SUBTREE`, which turns off the normal requirement that an entire container be checkpointed as a whole. Checkpointing just a subtree is a bit riskier than checkpointing a full container because it is harder to ensure that all needed resources have been saved. The `logfd` parameter is file descriptor open for writing; the kernel will write relevant logging information there. There are vast numbers of possible ways for a checkpoint to fail; the log file is intended to help users figure out what is happening when a checkpoint refuses to work. If logging is not desired, `logfd` can be `-1`. 

The set of processes to be checkpointed should be frozen prior to the call to `checkpoint()`. One exception to that rule is a process running in `checkpoint()` itself; this exception allows processes to checkpoint themselves. 

Internally, the checkpointing process is implemented as a two-phase operation: 

  * The kernel traverses the tree of processes and "collects" every object which is to be a part of the checkpoint image. Essentially, "collecting" means building a hash table with an entry for every process, every open file, every virtual memory area, every open socket, etc. which must be saved. Scanning the tree in this way helps the kernel to abort the checkpoint process early if something which cannot be checkpointed is encountered. Just as importantly, the collecting process also lets the system track objects which have multiple references and ensure that they are only written to the image file once. 

  * The second pass then iterates over the collected objects and causes each to be serialized and written to the image file. 

Once this is done, the checkpoint is finished. The just-checkpointed processes can either go on with their business or be killed, depending on the reason for the checkpoint. 

These two phases are reflected in the changes made to the lower levels of the system. For example, the none-too-svelte `file_operations` structure gains two new operations: 

```
int collect(struct ckpt_ctx *ctx, struct file *filp);
        int checkpoint(struct ckpt_ctx *ctx, struct file *filp);
```

The `collect()` operation should identify every object which must be saved, eventually passing each to `ckpt_obj_collect()` (or one of several higher-level interfaces) for tracking. Later, a call to `checkpoint()` is made to request that the given `filp` be serialized for saving to the checkpoint image. Similar methods have been added to a number of other structure types, including `vm_operations_struct` and `proto_ops`. 

The serialization process requires copying data from kernel data structures into a series of special structures intended to be written to the image file. So, for example, a file descriptor finds its way from `struct fdtable` into one of these: 

```
struct ckpt_hdr_file_desc {
    	struct ckpt_hdr h;
    	__s32 fd_objref;
    	__s32 fd_descriptor;
    	__u32 fd_close_on_exec;
        } __attribute__((aligned(8)));
```

Doing this copy requires a 75-line function which grabs the requisite information and very carefully checks that everything can be checkpointed successfully. In this case, the presence of locks on the file or an owner (to be notified with `SIGIO`) will cause the checkpoint to fail. In the absence of such roadblocks, the completed structure is handed to the checkpoint code for saving to the image file. 

This serialization process is one of the scarier parts of the whole checkpoint/restart concept. Any changes to `struct fdtable` will almost certainly break this serialization, quite possibly in ways which will not be detected until some user runs into a problem. Even if a VFS developer cared about checkpointing, they might not think to look at the code in `checkpoint/files.c` to see if anything might require changing. Similar dependencies are created for every other kernel data structure which must be checkpointed. The whole setup looks like it could be a little fragile; keeping it working is almost certain to require significant ongoing maintenance. 

#### Restarting

On the restart side, the application performing the restart is first expected to create a set of processes to be animated with the checkpointed information. That creation will be done with the much-reviewed "extended `clone()`" system call, which, in this iteration, looks like: 

```
int eclone(u32 flags_low, struct clone_args *cargs, int cargs_size,
    	       pid_t *pids);
```

With `eclone()`, the processes can be created with specific `pids` and with an extended set of flags. 

Once the process hierarchy exists, the `restart()` system call can be used: 

```
int restart(pid_t pid, int fd, unsigned long flags, int logfd);
```

The checkpoint image found at `fd` will be restored into the process hierarchy starting at `pid`. Once again, `logfd` can be used to gain information on how the process went. There are a number of `flags` defined: `RESTART_TASKSELF` (a single task is being restarted on top of the process calling `restart()`), `RESTART_FROZEN` (causes the restarted processes to be left frozen at the end), `RESTART_GHOST` (appears to be a debugging feature), `RESTART_KEEP_LSM` (restore security labels too), and `RESTART_CONN_RESET` (force the closing of open sockets). On a successful return from `restart()`, the process hierarchy should be ready to go. 

Once again, restart requires support at the lower levels of the kernel. So our long-suffering `file_operations` structure gains another function: 

```
int restore(struct ckpt_ctx *, struct ckpt_hdr_file *);
```

This function (along with its analogs elsewhere in the kernel) is charged with reanimating the given object from the checkpoint file. 

#### Security

It is not hard to imagine that these new system calls could have any of a number of security-related consequences, so it is surprising to see that, in the current implementation, both `checkpoint()` and `restart()` are unprivileged operations. This decision was made deliberately, with the idea of forcing the developers to think about security issues from the outset. 

The biggest potential problem with `checkpoint()` is probably information disclosure. To avoid this problem, `checkpoint()` is only able to checkpoint processes which the caller would be able to call `ptrace()` on. So there should be no way for a hostile user to gain information from a checkpoint image which would not be available anyway. 

The restart side is a little more frightening; it allows the caller to load vast amounts of potentially arbitrary data into kernel data structures. This risk is, one hopes, mitigated by causing all operations to be done in the context of the calling process. If the caller cannot open a file directly, that file cannot be opened via a corrupted checkpoint image either. Doing things this way will break certain use cases, such as checkpointing a setuid program which has since dropped its privileges, but there is probably no way to make that case work securely for unprivileged users. 

For an added challenge, the checkpoint/restart developers have also implemented the checkpointing of security labels on objects. By default, these labels will not be used during the restart process, but the `RESTART_KEEP_LSM` flag can change that. Again, the labels are created in the context of the calling process, so the active security module _should_ prevent the attachment of labels which compromise the security of the system. 

Even with these measures in place, one still has to wonder about the security of the process as a whole. The kernel is populating a wide array of data structures from input which may be under the control of a hostile user; it is not hard to imagine that, somewhere in tens of thousands of lines of checkpoint/restart code, an important check has not been made. Perhaps as a result of this concern, the patch set adds a sysctl knob which can be set to disallow unprivileged checkpoint/restart operations. 

#### Where things stand

According to the most recent patch posting: 

This one is able to checkpoint/restart screen and vnc sessions, and live-migrate network servers between hosts. It also adds support for x86-64 (in addition to x86-32, s390x and powerpc). 

So the patch set appears to be sufficiently functional to be minimally useful. There are, however, a lot of things which can stil prevent the creation of a successful checkpoint; they are summarized on [this page](<http://ckpt.wiki.kernel.org/index.php/Checklist>). Problem areas include private filesystem mounts, network sockets in some states, open-but-unlinked files, use of any of the file event notification interfaces, open files on network or FUSE filesystems, use of netlink, `ptrace()`, asynchronous I/O, and more. There are patches in the works for some of these problems; others look hard. 

As of this writing, there has been no response to the developers' request for inclusion in the -mm kernel. In the past, there have been concerns about how much work would be required to finish the job. Over the last year, much of that work is done, but checkpoint/restart looks like a job which is never truly finished. It's mostly a matter of whether what has been done so far appears to be good enough for real work, and whether the maintenance cost of this code is deemed to be worth paying.
