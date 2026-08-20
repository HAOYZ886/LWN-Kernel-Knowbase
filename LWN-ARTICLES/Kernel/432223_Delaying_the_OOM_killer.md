---
title: Delaying the OOM killer
url: https://lwn.net/Articles/432223/
date: "March 9, 2011"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "By Jonathan Corbet March 9, 2011"
---

By **Jonathan Corbet**  
March 9, 2011

The out-of-memory (OOM) killer is charged with killing off processes in response to a severe memory shortage. It has been the source of considerable discussion and numerous rewrites over the years. Perhaps that is inevitable given its purpose; choosing the right process to kill at the right time is never going to be an easy thing to code. The extension of the OOM killer into control groups has added to its flexibility, but has also raised some interesting issues of its own. 

Normally, the OOM killer is invoked when the system as a whole is catastrophically out of memory. In the control group context, the OOM killer comes into play when the memory usage by the processes within that group exceeds the configured maximum and attempts to reclaim memory from those processes have failed. An out-of-memory situation which is contained to a control group is bad for the processes involved, but it should not threaten the rest of the system. That allows for a little more flexibility in how out-of-memory situations are handled. 

In particular, it is possible for user space to take over OOM-killer duties in the control group context. Each group has a control file called `oom_control` which can be used in a couple of interesting ways: 

  * Writing "1" to that file will disable the OOM killer within that group. Should an out-of-memory situation come about, the processes in the affected group will simply block when attempting to allocate memory until the situation improves somehow. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

  * Through the use of a special `eventfd()` file descriptor, a process can use the `oom_control` file to sign up for notifications of out-of-memory events (see [Documentation/cgroups/memory.txt](<https://lwn.net/Articles/432224/>) for the details on how that is done). That process will be informed whenever the control group runs out of memory; it can then respond to address the problem. 

There are a number of ways that this user-space OOM killer can fix a memory issue that affects a control group. It could simply raise the limit for that group, for example. Alternatives include killing processes or moving some processes to a different control group. All told, it's a reasonably flexible way of allowing user space to take over the responsibility of recovering from out-of-memory disasters. 

At Google, though, it seems that it's not quite flexible enough. As has been widely reported, Google does not have very many machines to work with, so the company has a tendency to cram large numbers of tasks onto each host. That has led to an interesting problem: what happens if the user-space OOM killer is, itself, so starved for memory that it is unable to respond to an out-of-memory condition? What happens, it turns out, is that things just come to an unpleasant halt. 

Google operations is not overly fond of unpleasant halts, so an attempt has been made to find another solution. The outcome was [a patch from David Rientjes](<https://lwn.net/Articles/432226/>) adding another control file to the control group called `oom_delay_millisecs`. Like `oom_control`, it holds off the kernel's OOM killer in favor of a user-space alternative. The difference is that the administrator can provide a time limit for the kernel OOM killer's patience; if the out-of-memory situation persists after that much time, the kernel's OOM killer will step in and resolve the situation with as much prejudice as necessary. 

To David, this delay looks like a useful new feature for the memory control group mechanism. To Andrew Morton, instead, it looks like a kernel hack intended to work around user-space bugs, and he is not that thrilled by it. In [Andrew's view](<https://lwn.net/Articles/432230/>), if user space has set itself up as the OOM handler for a control group, it needs to ensure that it is able to follow through. Adding the delay looks like a way to avoid that responsibility which could have long-term effects: 

My issue with this patch is that it extends the userspace API. This means we're committed to maintaining that interface *and its behaviour* for evermore. But the oom-killer and memcg are both areas of intense development and the former has a habit of getting ripped out and rewritten. Committing ourselves to maintaining an extension to the userspace interface is a big thing, especially as that extension is somewhat tied to internal implementation details and is most definitely tied to short-term inadequacies in userspace and in the kernel implementation. 

Andrew would rather see development effort put into fixing any kernel problems which might be preventing a user-space OOM killer from doing its job. David, though, [doesn't see a way](<https://lwn.net/Articles/432232/>) to work without this feature. If it doesn't get in, Google may have to carry it separately; he predicted, though, that other users will start asking for it as usage of the memory controller increases. As of this writing, that's where the discussion stands.
