---
title: Kernel markers
url: https://lwn.net/Articles/245671/
date: "August 15, 2007"
category: Kernel markers; SystemTap
author: "By Jonathan Corbet August 15, 2007"
---

By **Jonathan Corbet**  
August 15, 2007

LWN's recent [look at SystemTap](<http://lwn.net/Articles/244536/>) noted that the patch set currently lacks a set of static probe points like that provided with DTrace. There are a few reasons for this difference. For example, the rate of change of the kernel code base would make the maintenance of a large set of probe points difficult, especially given that developers working on many parts of the code might not be particularly aware of - or concerned about - those points. But there is also the simple fact that the Linux kernel has no built-in mechanism for the creation of static probe points in the first place. 

There is, naturally, a patch which makes the creation of probe points possible; it is called [Linux kernel markers](<http://lwn.net/Articles/245382/>). This patch has been under development for some years. Its path into the mainline has been relatively rough, but there are signs that the worst of the roadblocks have been overcome. So perhaps a quick look at this patch is called for. 

With kernel markers, the placement of a probe point is easy: 

```
#include <linux/marker.h>
    
        trace_mark(name, format_string, ...);
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The `name` is a unique identifier which is used to access the probe; the documentation recommends a `subsystem_event` format, describing the subsystem in which the probe is found and the event which is being traced. For example: in a part of the patch which instruments the block subsystem, a probe placed in `elv_insert()`, which inserts a request into its proper location in the queue, is named `blk_request_insert`. The format string describes the remaining arguments, each of which will be some variable of interest at the time the trace point is hit. 

Code which wants to hook into a trace point must call: 

```
int marker_probe_register(const char *name, const char *format,
    			      marker_probe_func *probe, void *pdata);
```

Here, `name` is the name of the trace point, `format` is the format string describing the expected parameters from the trace point (it must match the format string provided when the trace point was established), `probe()` is the function to call when the trace point is hit, and `pdata` is a private data value to pass to `probe()`. The `probe()` function will have this prototype: 

```
void (*probe)(const struct __mark_marker *mdata, void *pdata,
                      const char *format, ...);
```

The `mdata` structure includes the name of the trace point, if need be, along with a formatted version of the arguments. The arguments themselves are passed after the format string. 

Registration of a marker does not, yet, set up the `probe()` function to be called. First, the marker must be armed with: 

```
int marker_arm(const char *name);
```

Once the marker has been armed, `probe()` will be called every time execution arrives at the given trace point. 

When probe points are no longer of interest, they can be shut down with: 

```
int marker_disarm(const char *name);
        void marker_probe_unregister(const char *name);
```

Calls to `marker_arm()` will nest - if a given marker has been armed three times, then three `marker_disarm()` calls will be required to turn it off again. 

Internally, there are a lot of details to the management of markers. The code at the actual trace point, in the end, looks much like one would expect: 

```
if (marker_is_armed) {
            preempt_disable();
    	(*probe)(...);
    	preempt_enable();
        }
```

In reality, it is not quite so simple. Getting marker support into the kernel requires that the runtime impact of kernel markers be as close to zero as possible, especially when the marker is not armed. A common use case for markers is to investigate performance problems on systems running in production, so they have to be present in production kernels without causing performance problems themselves. Adding a test-and-jump operation to a kernel hot path will always be a hard sell; the cache effects of referencing a set of global marker state variables could also be significant. 

To get around this problem, the marker code comes with a separate patch called [immediate values](<http://lwn.net/Articles/245381/>). In the architecture-independent implementation, an immediate value just looks like any other shared variable. The purpose of immediate values, though, is to provide variables with the assumption that they will be frequently read but infrequently changed, and that the read operations must have the lowest impact possible. So, in an architecture-specific implementation (which only exists for i386 at the moment), changing an immediate value actually patches any code which reads the value. To say that the details of doing this sort of patching safely are ugly would be to understate the point. But Mathieu Desnoyers has dealt with those details, and nobody else need look at the resulting code. 

Through the use of immediate values, the code inserted by `trace_mark()` can query the setting of a trace point without generating a memory reference at all; instead, that setting is stored directly in the inserted code. So there will be no potential for an expensive cache miss at the probe point. The patch also provides an `immediate_if()` construct which is intended to allow jumps to be patched directly into the code, eliminating the test altogether, but that functionality has not yet been implemented. Even without this feature, immediate values allow the creation of trace points whose runtime impact is very nearly zero, eliminating the most common objection to their existence. 

If and when this code is merged, the way will be clear for the creation of a set of well-defined trace points for utilities like SystemTap and [LTTng](<http://ltt.polymtl.ca/>). That, in turn, could make the internal operations of the kernel more visible to system administrators and others who are not necessarily well versed in how the kernel works. This sort of tracing ability has been on many users' wish lists for some time; they might just be, finally, getting close to having that wish fulfilled.
