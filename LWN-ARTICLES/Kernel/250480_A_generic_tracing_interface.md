---
title: A generic tracing interface
url: https://lwn.net/Articles/250480/
date: "September 19, 2007"
category: "Development tools-Kernel tracing; SystemTap; Tracing"
author: "By Jonathan Corbet September 19, 2007"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 19, 2007

Dynamic kernel tracing remains high on the wishlists presented by many Linux users. While much work has been done to create a powerful tracing capability, very little of that work has found its way into the mainline. The recent posting of one small piece of infrastructure may help to change that situation, though. 

The piece in question is the [trace layer](<http://lwn.net/Articles/249868/>) posted by David Wilder. Its purpose is to make it easy for a tracing application to get things set up in the kernel and allow the user to control the tracing process. To that end, it provides an internal kernel API and a set of control files in the debugfs filesystem. 

On the kernel side, a tracing module would set things up with a call to: 

```
#include <linux/trace.h>
    
        struct trace_info *trace_setup(const char *root, const char *name,
    			           u32 buf_size, u32 buf_nr, u32 flags);
```

Here, `root` is the name of the root directory which will appear in debugfs, `name` is the name of the control directory within `root`, `buf_size` and `buf_nr` describe the size and number of relay buffers to be created, and `flags` controls various channel options. The `TRACE_GLOBAL_CHANNEL` flag says that a single set of relay channels (as opposed to per-CPU channels) should be used; `TRACE_FLIGHT_CHANNEL` turns on the "flight recorder" mode where relay buffer overruns result in the overwriting of old data, and `TRACE_DISABLE_STATE` disables control of the channel via debugfs. 

The return value (if all goes well) will be a pointer to a `trace_info` structure for the channel. This structure has a number of fields, but the one which will be of most interest outside of the trace code itself will be `rchan`, which is a pointer to the relay channel associated with this trace point. 

When actual tracing is to begin, the kernel module should make a call to: 

```
int trace_start(struct trace_info *trace);
```

The return value follows the "zero or a negative error value" convention. Tracing is turned off with: 

```
int trace_stop(struct trace_info *trace);
```

When the tracing module is done, it should shut down the trace with: 

```
void trace_cleanup(struct trace_info *trace);
```

Note that none of these entry points have anything to do with the placement or activation of trace points or the creation of trace data. All of that must be done separately by the trace module. So a typical module will, after calling `trace_start()`, set up one or more kprobes or activate a static kernel marker. The probe function attached to the trace points should do something like this: 

```
rcu_read_lock();
        if (trace_running(trace)) {
            /* Format trace data and output via relay */
        }
        rcu_read_unlock();
```

Additionally, if the `TRACE_GLOBAL_CHANNEL` flag has been set, the probe function should protect access to the relay channel with a spinlock. This protection may also be necessary in situations where an interrupt handler might be traced. 

In user space, the trace information will show up under `/debug/root/name`, where `debug` is the debugfs mount point, and `root` and `name` are the directory names passed to `trace_setup()`. The file `state` can be read to get the current tracing state; an application can write `start` or `stop` to this file to turn tracing on or off. The file `trace0` is the relay channel where tracing data can be read; on SMP systems with per-CPU channels there will be additional files (`trace1`...) for additional processors. The file `dropped` can be read to see how many trace records (if any) have been dropped due to buffer-full conditions. 

All told, it is not a particularly complicated bit of code. Perhaps the most significant feature of this patch is that it is part of the infrastructure created and used by the SystemTap project. Getting this code into the mainline will make it that much easier for distributors to provide well-supported tracing facilities to their users. And that, in turn, should make users happy and give analysts one less thing to complain about.
