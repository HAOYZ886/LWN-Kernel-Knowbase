---
title: User events — but not quite yet
url: https://lwn.net/Articles/889607/
date: "April 18, 2022"
category: "BPF-Tracing; Tracing"
author: "By Jonathan Corbet April 18, 2022"
---

By **Jonathan Corbet**  
April 18, 2022

The ftrace and perf subsystems provide visibility into the workings of the kernel; by activating existing tracepoints, interested developers can see what is happening at specific points in the code. As much as kernel developers may resist the notion, though, not all events of interest on a system happen within the kernel. Administrators will often want to look inside user-space processes as well; they would be even happier with a mechanism that allows the simultaneous tracing of events in both the kernel and user space. The [user-events subsystem](<https://lwn.net/ml/linux-kernel/20220118204326.2169-1-beaub@linux.microsoft.com/>), developed by Beau Belgrave and added during the 5.18 merge window, promises that capability, but users will almost certainly have to wait another cycle to gain access to it. 

Kernel tracepoints are hooks at specific locations in the code. They are designed to add as little overhead as possible when they are not active, which is the case most of the time. When a tracepoint is activated, it produces a stream of structured data specific to the event being monitored; user space can read that data via a number of different interfaces. By turning on just the tracepoints of interest, user space can collect the data needed to analyze a specific situation without slowing down the kernel overall. 

#### The user-events ABI

Belgrave's user-space equivalent to kernel-space tracepoints, merged for 5.18, requires a bit more work to support, though libraries provided in the future may ease some of that burden. The first step is to open a new file added to the tracefs kernel filesystem: 

```
/sys/kernel/debug/tracing/user_events_data
```

A program then needs to register each event that it wishes to make available to the system. That is done by filling out this structure: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

```
struct user_reg {
            u32 size;
            u64 name_args;
            u32 status_index;
            u32 write_index;
        };
```

The first two members are input parameters, while the last two are set by the kernel. The `size` parameter should just be the size of the `user_reg` structure itself; this helps to ensure compatibility if the structure grows in future kernel releases. The event itself is described by `name_args`, which is a pointer to a string; it uses a special format added with this patch set. The first token is the name of the event; the rest of the line describes the data reported for that event. So an event that reports an integer named `level` and a 20-character string named `badness` could be described as: 

```
my-event u32 level; char[20] badness
```

This structure is then registered with an [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) operation on the previously opened `user_events_data` file, using the `DIAG_IOCSREG` command. On successful registration, the kernel will store two index values in `status_index` and `write_index`, the use of which will be described below. 

Once the event is registered, it will show up in tracefs under the `user_events` subsystem. That means it can be activated, and its data collected, using any of the usual user-space tools. But to get there, the application must still provide that data when the time comes. 

To do that, the program should open the other new tracefs file as well: 

```
/sys/kernel/debug/tracing/user_events_status
```

That file should then be mapped into the program's address space with an [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) call. 

Like its kernel counterpart, the user-events mechanism has been designed to minimize its overhead when nobody is interested in the events. So the program implementing the events will only want to provide the data if it has been requested. The `user_events_status` file that was just mapped above will contain a byte of data indicating whether the event is active or not; its index will be the value stored in the `status_index` field during registration. If that byte is zero, the event is not active and the program should not output any data; that is expected to be the case most of the time. 

When somebody attaches to the event, the associated byte will no longer read as zero. It is, in fact, a bitmap giving information about how the event has been attached; one bit corresponds to ftrace, while another is for perf. When the program sees that non-zero byte, it should write the data associated with the event to the `user_events_data` file opened at the beginning. The first four bytes of the written data should be the value the kernel stored in `write_index` at registration time; the rest will be the data as described. Typically, a [`writev()`](<https://man7.org/linux/man-pages/man2/readv.2.html>) call will be needed to assemble the requisite bits. 

That describes the bulk of the API. More information can be found in [this documentation commit](<https://git.kernel.org/linus/864ea0e10cc9>) and [this sample program](<https://git.kernel.org/linus/c57eb4781509>). There is also, inevitably, a way to attach BPF programs to user events, but that feature is not described in detail in the documents. 

#### Concerns

After this code was merged, [Linux Trace Toolkit](<https://lttng.org>) (LTTng) developer Mathieu Desnoyers posted [some criticisms](<https://lwn.net/ml/linux-kernel/2059213643.196683.1648499088753.JavaMail.zimbra@efficios.com/>) of the new interface. The byte-based status mechanism struck him as inefficient; providing a single bit for each event would allow for a more compact representation and, thus, better cache utilization. The multiple bits of information indicating how the events had been attached to have no real value to the application being traced, which should produce the same data regardless. 

He had some other concerns as well. If the page(s) containing the data to be written for an event are forced out of memory, the resulting page fault will cause `writev()` to fail and, absent active countermeasures, the event data will be lost. The mechanism as a whole is built around access to tracing data via the kernel; it will only add overhead when purely user-space tracers (such as LTTng) are in use. There were a number of implementation concerns as well. 

Desnoyers also brought the facility to the attention of BPF maintainer Alexei Starovoitov, who had been unaware of it. He [was not happy](<https://lwn.net/ml/linux-kernel/CAADnVQK=GCuhTHz=iwv0r7Y37gYvt_UBzkfFJmNT+uR0z+7Myw@mail.gmail.com/>) with what he saw; he called for the BPF mechanism to be removed immediately: ""It's a hard Nack to add a bpf interface to user_events"". He has reiterated that position in subsequent discussions. 

Belgrave quickly [posted a patch](<https://lwn.net/ml/linux-kernel/20220329173051.10087-1-beaub@linux.microsoft.com/>) removing the BPF feature, as requested. But it looks like that will not be enough for this feature to be enabled in 5.18. Tracing maintainer Steven Rostedt [stated his agreement](<https://lwn.net/ml/linux-kernel/20220329154532.4833d16d@gandalf.local.home/>) with Desnoyers, saying that he is considering [marking the whole mechanism as "broken"](<https://lwn.net/ml/linux-kernel/20220329222514.51af6c07@gandalf.local.home/>) so that the issues can be resolved. It is conceivable that Belgrave could address all of the concerns in this development cycle, but it is unlikely; that sort of work is not meant to go into the mainline after the merge window closes. So, chances are, users will have to wait until 5.19 for access to the new user-events tracing mechanism.
