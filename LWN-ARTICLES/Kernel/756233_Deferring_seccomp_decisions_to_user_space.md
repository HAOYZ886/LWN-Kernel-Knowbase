---
title: Deferring seccomp decisions to user space
url: https://lwn.net/Articles/756233/
date: "June 2, 2018"
category: "Security-seccomp"
author: "By Jonathan Corbet June 2, 2018"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 2, 2018

There has been a lot of work in recent years to use BPF to push policy decisions into the kernel. But sometimes, it seems, what is really wanted is a way for a BPF program to punt a decision back to user space. That is the objective behind [this patch set](<https://lwn.net/Articles/756167/>) giving the [secure computing (seccomp)](<http://man7.org/linux/man-pages/man2/seccomp.2.html>) mechanism a way to pass complex decisions to a user-space helper program. 

Seccomp, in its most flexible mode, allows user space to load a BPF program (still "classic" BPF, not the newer "extended" BPF) that has the opportunity to review every system call made by the controlled process. This program can choose to allow a call to proceed, or it can intervene by forcing a failure return or the immediate death of the process. These seccomp filters are known to be challenging to write for a number of reasons, even when the desired policy is simple. 

Tycho Andersen, the author of the "seccomp trap to user space" patch set, sees a number of situations where the current mechanism falls short. His scenarios include allowing a container to load modules, create device nodes, or mount filesystems — with rigid controls applied. For example, creation of a `/dev/null` device would be allowed, but new block devices (or almost anything else) would not. Policies to allow this kind of action can be complex and site-specific; they are not something that would be easily implemented in a BPF program. But it might be possible to write something in user space that could handle decisions like these. 

To enable this, Andersen's patch set adds a new return type for BPF programs (`SECCOMP_RET_USER_NOTIF`) that will cause the program making the call to be blocked while information about the call is sent to user space. A controlling program wanting to receive these notifications (and make decisions) must open a file descriptor by setting the `SECCOMP_FILTER_FLAG_GET_LISTENER` flag when loading the filter program. The returned file descriptor can then be polled for events; reading from it will return the next available notification signaled by the BPF filter. 

Notifications, when read, are encoded in this structure: 

```
struct seccomp_notif {
    	__u64 id;
    	pid_t pid;
    	struct seccomp_data data;
        };
```

The returned `id` is a unique number identifying this event, `pid` is the ID of the process that triggered the notification, and `data` is the `seccomp_data` structure that was given to the BPF program describing the system call in progress: 

```
struct seccomp_data {
    	int   nr;                   /* System call number */
            __u32 arch;                 /* AUDIT_ARCH_* value
                                           (see <linux/audit.h>) */
            __u64 instruction_pointer;  /* CPU instruction pointer */
            __u64 args[6];              /* Up to 6 system call arguments */
        };
```

The user-space program can then meditate on whatever it is that the controlled program wishes to do. Note that the behavior of user notifications is similar to `SECCOMP_RET_ERRNO`, in that the system call itself will not be invoked in the context of the controlled process. So if the controlling process wants the system call to run in some form, it must do the work in its own context. When it has reached a decision (and done any needed work), it communicates that back to the kernel by filling in a `seccomp_notif_resp` structure and writing it back to the notification file descriptor: 

```
struct seccomp_notif_resp {
    	__u64 id;
    	__s32 error;
    	__s64 val;
        };
```

The `id` value must match that found in the original notification. `error` should be either zero or a negative error code; in the latter case, it will be negated and used as an error return from the system call that created the notification in the first place. If `error` is zero, then that system call will return successfully with `val` as its return value. 

As a somewhat experimental addition, the final patch in the series adds two fields to the `seccomp_notif_resp` structure: 

```
__u8 return_fd;
    	__u32 fd;
```

These fields allow the control program to provide a file descriptor to be used as the return value from the system call; if `return_fd` is nonzero, `fd` will be passed to the controlled program. As Andersen notes, this mechanism will only work for system calls that are expected to return a file descriptor in the first place, but it's a starting point. 

The protocol for the communication between the kernel and the control program has been the topic of some discussion in the past; in its current form, it will be difficult to extend when new features are (inevitably) added. Reviewers in the past have suggested using the netlink protocol instead, but that involves more complexity than the current implementation. Whether those reviewers will insist on that change before this code can be merged remains to be seen. 

Overall, this patch series is another step in an interesting set of changes that has been taking place. The boundary between the kernel and user space was once a hard and well-defined line described by the system-call interface. Increasingly, developers are working to make it possible for users to move functionality across that line in both directions, both putting policy into the kernel with BPF programs or moving it out with various types of user-space helpers. As the computing environment changes, it seems that this flexibility will be needed to ensure that Linux stays relevant.
