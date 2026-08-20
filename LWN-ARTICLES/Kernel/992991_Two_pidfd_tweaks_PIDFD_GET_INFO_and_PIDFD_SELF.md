---
title: "Two pidfd tweaks: PIDFD_GET_INFO and PIDFD_SELF"
url: https://lwn.net/Articles/992991/
date: "October 16, 2024"
category: "Releases-6.13; pidfd"
author: "By Jonathan Corbet October 16, 2024"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 16, 2024

The pidfd mechanism, which uses file descriptors to refer to processes in an unambiguous and race-free way, [was first introduced](<https://lwn.net/Articles/773459/>) in 2018. Since then, the interface has [gained a number of new features](<https://lwn.net/Articles/794707/>), but development has slowed over time as the interface has matured. There are, however, a couple of patches in circulation that are meant to make working with pidfds simpler in some situations. 

#### Information from pidfds

There are a number of things that can be done with a pidfd, including sending a signal to the process it represents or waiting for the process to exit. One thing that is missing, though, is the ability to acquire information about the process. The only way to do that in current kernels is to turn the pidfd into a regular process ID, then use that PID to open files under `/proc` and read the needed information there. This solution works, but it is slow, multi-step, and racy, since the process in question might disappear in the middle of the sequence. It also requires that `/proc` be present. It seems like there should be a better way. 

The current contender for that better way is [this patch](<https://lwn.net/ml/all/20241008121930.869054-1-luca.boccassi@gmail.com>) from Luca Boccassi. It adds a new [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) operation for pidfds called `PIDFD_GET_INFO`: 

```
size = ioctl(pidfd, PIDFD_GET_INFO, &info);
```

The `info` pointer is where the returned information will be put; it is currently defined as: 

```
struct pidfd_info {
    	/* Let userspace request expensive stuff explictly, and let the kernel
    	 * indicate whether it knows about it. */
    	__u64 request_mask;
    	__u64 cgroupid;
    	__u32 pid;
    	__u32 tgid;
    	__u32 ppid;
    	__u32 ruid;
    	__u32 rgid;
    	__u32 euid;
    	__u32 egid;
    	__u32 suid;
    	__u32 sgid;
    	__u32 fsuid;
    	__u32 fsgid;
    	__u32 spare0[1];
        };
```

The `request_mask` field is an input parameter to the operation that controls options for the information request; the remaining parameters are used to return the desired information: the process ID (`pid`), thread-group ID (`tgid`), parent-process ID (`ppid`), the real user and group IDs (`ruid` and `rgid`), the effective user and group IDs (`euid` and `egid`), saved user and group IDs (`suid` and `sgid`), and filesystem user and group IDs (`fsuid` and `fsgid`). The ID of the control group containing the process is returned in `cgroupid`, but only if the `PIDFD_INFO_CGROUPID` bit has been set in `request_mask` (because this value is a bit more expensive to obtain). That same bit will be set in `request_mask` on return if `cgroupid` is valid. 

One can easily imagine that the list of data that one might want to obtain will grow in the future. As is the case with a number of recently added kernel features, the kernel checks the size of the `pidfd_info` structure passed from user space to detect older or newer versions of the interface. Providing new features will require adding new fields to the structure, which will change its size, allowing a mismatch to be detected. The kernel will not return more information than the passed-in structure can hold, so new fields can be added in the future without breaking binary compatibility. 

While system calls typically require user space to pass the size of the structure explicitly to the kernel so that this test can be made, the use of `ioctl()` makes things just a bit easier. As is routinely done with `ioctl()` commands, the size of the structure is encoded directly into the `PIDFD_GET_INFO` macro, so it is passed implicitly to the kernel when that operation is requested. 

This patch has been through nine revisions as of this writing, and seems likely to see at least one more. There does not appear to be any opposition to its objective or the interface provided, though, so it can be expected to land in the mainline before too long. 

#### Self-references

Pidfds are meant to allow one process to maintain a handle on another, and there are several system calls that allow that handle to be used to act on another process in some way. Sometimes, though, a process would like to use one of those system calls to act on itself. System calls that take a process ID as an argument typically interpret a value of zero as referring to the calling process, but there is no equivalent notation for the pidfd system calls. As a result, any process wanting to use one of those system calls must first obtain a pidfd for itself with [`pidfd_open()`](<https://man7.org/linux/man-pages/man2/pidfd_open.2.html>). 

Most of the time. this omission does not matter; for most pidfd-oriented system calls, there is an equivalent that takes a process ID instead that can be used. The exception is [`process_madvise()`](<https://man7.org/linux/man-pages/man2/process_madvise.2.html>), which has no non-pidfd equivalent — a fact that [drove some discussion](<https://lwn.net/Articles/810076/>) while the call was being developed. So, if a process wants to call `process_madvise()` on itself, there is no alternative to obtaining its own pidfd. So there is no alternative to obtaining a pidfd for one's self if the objective is to use `process_madvise()` on the calling process. 

Lorenzo Stoakes [proposed](<https://lwn.net/ml/all/cover.1727644404.git.lorenzo.stoakes@oracle.com>) to change that situation with a seemingly simple patch allowing the pidfd argument of the relevant system calls to be specified as `PIDFD_SELF`. The call would then act on the calling process, with no need to open and manage a separate pidfd. The idea seems straightforward and uncontroversial, but it turns out there is one little catch. 

Florian Weimer [asked](<https://lwn.net/ml/all/87ttdxl9ch.fsf@oldenburg.str.redhat.com>) whether `PIDFD_SELF` would refer to the current process (which means referring to the thread-group leader within the process), or the thread that is actually making the call. For some system calls, that distinction matters; the outcome will be different depending on which interpretation is used. Stoakes [had intended](<https://lwn.net/ml/all/42df57ac-d89c-4111-a04d-290dd2197573@lucifer.local>) for `PIDFD_SELF` to refer to the thread-group leader, but Christian Brauner [argued convincingly](<https://lwn.net/ml/all/20240930-verbiegen-zinspolitik-cafb730c3c84@brauner>) that the current-thread interpretation is more likely to be what the caller wants. So the patch set had to change. 

The [updated version](<https://lwn.net/ml/all/cover.1728578231.git.lorenzo.stoakes@oracle.com>) creates two new special pidfd values: `PIDFD_SELF_THREAD`, which will refer to the calling thread, and `PIDFD_SELF_THREAD_GROUP`, which refers to the thread-group leader. While those names are made available to user space, there are also aliases for those names (`PIDFD_SELF` and `PIDFD_SELF_PROCESS`, respectively) that are what user-space programs are actually expected to use. In summary, the default (`PIDFD_SELF`) has changed to mean the current thread, but the group-leader semantics are available as well. 

With that issue resolved, there do not appear to be any remaining concerns about the functionality provided by this patch series. Barring some sort of surprise, `PIDFD_SELF` and `PIDFD_SELF_PROCESS` seem likely to make an appearance in the 6.13 release.
