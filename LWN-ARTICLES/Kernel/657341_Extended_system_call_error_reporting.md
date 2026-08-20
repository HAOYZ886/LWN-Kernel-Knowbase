---
title: Extended system call error reporting
url: https://lwn.net/Articles/657341/
date: "September 16, 2015"
category: "Error codes; Messages; User-space API-Error reporting"
author: "By Jonathan Corbet September 16, 2015"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 16, 2015

The interface between the kernel and user space is, in places, surprising in its complexity. There are numerous tasks that involve passing detailed information about hardware configurations, process state, and more, in either direction. When something goes wrong, though, that communication channel narrows to a single integer error code, often making it difficult for developers to figure out what is going on. There have been various proposals for widening the error-reporting interface in the past; the [latest proposal](<https://lwn.net/Articles/657147/>), from Alexander Shishkin, may not get any further than its predecessors, but it does show that there is ongoing interest in the problem. 

As an example, consider the `VIDIOC_S_FMT` `ioctl()` call, provided by the media subsystem. Its job is to set the format of images returned to user space from a capture device (such as a camera). There is a mind-boggling variety of possible image formats out there, so the [format description](<http://linuxtv.org/downloads/v4l-dvb-apis/pixfmt.html>) passed to the kernel from user space contains a number of complex, interrelated parameters. There are a lot of ways such a description can go wrong — and that's before the vagaries of specific driver implementations are taken into account. Should there be a problem, though, the only thing user space is likely to know is that the `VIDIOC_S_FMT` call returned `EINVAL`. The kernel, of course, knows what it was objecting to, but there is no way to communicate that knowledge to user space. 

Fixing that problem is not easy; the `errno` mechanism is clearly inadequate, but it is set in stone by several decades of Unix tradition and cannot be changed without breaking applications. So any extended error information must be carried by a new channel that can be ignored by unaware applications. The addition of error information to the kernel must also be done carefully, so as to avoid slowing down kernel hot paths or clogging the source with an overwhelming set of error messages. Alexander's patch attempts to meet all of these criteria. 

How the mechanism works is best illustrated with some examples. In his patch set, Alexander targets the [`perf_event_open()`](<http://man7.org/linux/man-pages/man2/perf_event_open.2.html>) system call; it takes a `perf_event_attr` structure as a parameter. That structure has a vast and growing set of parameters describing the events to be captured and, correspondingly, there are a lot of ways in which things can go wrong. 

#### Describing errors

The first step is to create a structure that describes an error site — a place where an error is detected and passed back to user space. That structure should include a field called `site` that holds an `ext_err_site` structure; beyond that, it can contain any information needed to fully report the error to user space. In the perf case, that structure looks like this: 

```
#include <linux/exterr.h>
    
        struct perf_ext_err_site {
    	struct ext_err_site	site;
    	const char		*attr_field;
        };
```

The `attr_field` member is meant to hold the name of the field in `struct perf_event_attr` that generated the error. 

Then, it is necessary to define a function that can turn any extra information in this structure into a string to be passed back to user space. The perf version is: 

```
static char *perf_exterr_format(void *site)
        {
    	struct perf_ext_err_site *psite = site;
    
    	return kasprintf(GFP_KERNEL, "\t\"attr_field\": \"%s\"\n",
    			 psite->attr_field);
        }
```

Note that this function returns a dynamically allocated string; the extended error infrastructure will free that string when it is no longer needed. 

With these two pieces in place, it is possible to define an "error domain" that handles a specific class of errors — perf errors in this case: 

```
DECLARE_EXTERR_DOMAIN(perf, perf_exterr_format);
```

The actual reporting of error information is done by way of a rather frightening bit of macro magic called `ext_err()`. Real users will almost certainly wrap it, though; this is how it is done in the perf code: 

```
#define perf_err(__code, __attr, __msg)				\
    	({ /* make sure it's a real field before stringifying it */	\
    	    struct perf_event_attr __x; (void)__x.__attr;		\
    	    ext_err(perf, __code, __msg, 				\
    	        .attr_field = __stringify(__attr));			\
    	})
```

The parameters to `ext_err()` are the name of the domain defined above, the (negative) error code, a message to be reported to user space, and a set of initialization strings to initialize the rest of the error-site structure. In this case, the final parameter to `ext_err()` sets the `attr_field` of the `perf_ext_err_site` structure to the name of the erroneous attribute. See [this patch](<https://lwn.net/Articles/657346/>) for an actual invocation of the `perf_err()` macro. 

There are a couple of other important details. One is that the `EXTERR_MODNAME` symbol must be set separately before calling `ext_err()`: 

```
#define EXTERR_MODNAME	"perf"
```

The other is that `ext_err()` returns a value, which is a modified version of the error code passed into it. This code can be thought of as an index into an array of `ext_err_site` structures describing every extended error known to the kernel. The normal way to return the error code to user space will then be through a line like: 

```
return ext_err_errno(code);
```

The modified code from `ext_err()` must not be returned directly to user space, since applications will have no idea what it means. On the other hand, the original error code should not be returned without calling `ext_err_errno()`; that call is the one that causes the kernel to remember the extended error information. In short, there is a new `task_struct` field called `ext_err_code`; the call to `ext_err_errno()` causes the special error code to be placed into that field. If an ordinary (non-extended) error code is passed to `ext_err_errno()`, the right thing will happen, so it is safe to use in situations where a support code might return ordinary or extended error codes. 

#### The user-space side

At this point, the kernel is prepared to tell user space about an extended error, but the return value from the system call can still only be the ordinary `errno` value that it has always been. If the application wants to know more, it can make a call like: 

```
char message[SIZE];
    
        len = prctl(PR_GET_ERR_DESC, message, SIZE);
```

The return value is not just an ordinary message; it is a string in the [JSON](<http://json.org/>) format containing the file and line where the error was generated, the error code, the module name, the actual message, and any specific information added by the domain format function described above. The changes to the user-space `perf` tool duly include a new JSON parser to pick this message apart again. The `prctl()` call will clear the error information on the kernel side, so a second call will return no data. 

The patch set has, thus far, not seen much in the way of review comments. In the end, the error-reporting issue is one that most developers recognize, but few feel up to trying to fix. So it is hard to say whether this attempt to widen the error-reporting channel from the kernel will meet with success or not. Ancient traditions are hard to change but, every now and then, somebody succeeds.
