---
title: The structured logging challenge
url: https://lwn.net/Articles/451365/
date: "July 12, 2011"
category: Kernel messages
author: "By Jonathan Corbet July 12, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 12, 2011

The debate over the concept of "[user-friendly disk names](<https://lwn.net/Articles/448505/>)" was rekindled this week with the posting of a new version of Nao Nishijima's [persistent device names patch](<https://lwn.net/Articles/450884/>). The disagreements over this particular feature remain; it is possible that the change will be merged regardless. At the core of this discussion, though, is a concept which goes beyond adding a user-specified name to specific devices; it's the bigger problem of getting structured data out of the kernel. 

After all these years, the main mechanism by which the kernel passes information to user space remains the lowly `printk()` function. It is, needless to say, a useful and flexible way of getting messages out, but it imposes almost nothing on the structure of those messages. That leads to all kinds of output like (from `drivers/net/de620.c`): 

```
printk(KERN_WARNING "%s: Thanks, I feel much better now!\n", dev->name);
```

or the famous message from `drivers/char/lp.c`: 

```
printk(KERN_INFO "lp%d on fire\n", minor);
```

System administrators should not be faulted for wondering what they should do in response to messages like these. 

There have been some changes to impose structure on `printk()` output, starting with the addition of a marker for the severity level of each message. It still is not hard to find `printk()` calls without severity levels, though; actually enforcing the use of these markers has proved hard to do. A bit more structure is added by `dev_printk()` (and variants like `dev_err()`), but the use of these functions is even less universal. 

The lack of structure means that there is little consistency between messages; any two network drivers will almost certainly print different things to indicate the same situation. Kernel messages can also vary over time; messages emitted by `printk()` are also not normally seen as part of the kernel ABI, despite the fact that changing them can break scripts that try to extract useful information from the system logs. So it's not surprising that, about one year ago, Andrew Morton [said](<https://lwn.net/Articles/392177/>): 

The kernel's whole approach to messaging is pretty haphazard and lame and sad. There have been various proposals to improve the usefulness and to rationally categorise things in way which are more useful to operators, but nothing seems to ever get over the line. 

Various people have tried to improve the situation in spots; the user-friendly disk names, by trying to attach a consistent name to devices, is one such attempt. The [netoops patch](<https://lwn.net/Articles/419845/>) from Google is another; it helps Google figure out why machines are crashing without requiring operators to actually dig through the logs. But these changes are far from an overall framework for structured data from the kernel. 

There have been a few attempts to make such frameworks over the years; all have fallen far short of making it into the kernel. It's not hard to come up with plausible reasons for this failure. The amount of work required is huge, especially if one wants to add structure to the bulk of interesting communications from kernel space. Developers like `printk()`; they are less likely to be enamored of some other interface which requires more work to use, is less flexible (by design) in its output, and which may well have to sit alongside the existing `printk()` logging. Coming up with a structured format which meets everybody's needs - and which will not have to be supplemented with a "version 2" format in the coming years - presents its own challenges. 

It must also be said that kernel developers, as a whole, see little value in standardized, structured kernel logging information. It will not help them to debug their kernels. The fact that a lot of users want this sort of feature is far from irrelevant to the development community, but experience has shown that a lack of developer interest can make it much harder to get changes merged - especially if those changes are wide-ranging and disruptive. 

If this problem is ever to be solved, it would seem that two things need to be found: a mechanism which looks like it could work and a motivation for kernel developers to accept it. The motivation can probably found in a combination of (1) their paychecks as customers continue to push for this capability and (2) the prospect of a continuing stream of _ad hoc_ patches adding structure to various corners without solving the real problem. But that leaves open the problem of finding a workable solution. 

Your editor has a half-baked thought on this matter based on the realization that the kernel already has a nice mechanism for passing structured data to user space. On almost any contemporary system, the `/dev` directory is managed by the `udev` daemon; `udev` works by receiving highly-structured messages from the kernel describing the coming and going of devices, changes in their configuration, firmware load requests, and more. It is an established protocol which enables sophisticated user-space responses to kernel events. Udev and the associated "uevent" mechanism had some early growing pains, but this code is now stable, functional, and almost universally used. Perhaps it's time for this mechanism to take on some new duties. 

Uevents work because the format is simultaneously structured and flexible; it can be extended when the need arises. The generation of events is almost entirely done automatically by the driver core; most driver authors need not do anything to cause them to happen and, indeed, may not even know that this mechanism is operating underneath the hood. Driver authors don't have to make their own events; they would have to go out of their way to prevent them from happening. 

Logging of other types of events will probably require explicit support in the relevant kernel code; that is the part needing some extra thought. The creation of uevents by hand is a bit of a labor-intensive business; the relevant code tends to look like: 

```
retval = add_uevent_var(env, "ACTION=%s", action_string);
    	if (retval)
    		goto exit;
    	retval = add_uevent_var(env, "DEVPATH=%s", devpath);
    	if (retval)
    		goto exit;
    	retval = add_uevent_var(env, "SUBSYSTEM=%s", subsystem);
    	if (retval)
    		goto exit;
```

Clearly, any attempt to place this kind of code in every logging location is not going to get very far. What is needed is a useful set of helper functions. These functions, for maximum utility, would probably be tied fairly tightly to the underlying subsystems. Storage drivers could have functions to report block errors, device changes, and multipath connectivity changes. Network drivers would need to report events like carrier loss, excessive checksum errors, or duplicate MAC addresses. All kernel code could benefit from helpers to log allocation failures or failed assertions. In each case, the helper would standardize the format of the reported information while allowing the addition of information specific to the call site. 

The addition of a new set of logging functions would necessarily require changes to drivers to use those functions. So it would take time to achieve anything close to comprehensive coverage, and 100% coverage would never happen. But, then, we still don't have 100% coverage for the KERN_* severity markers. If this interface proved useful, one could imagine that the code paths of interest to enterprise distribution customers would be covered in relatively short order. 

But, then, there are probably several things fatally wrong with this idea; the structured logging problem will likely remain unsolved for some time yet. But the problem will not go away; if anything, the need to recognize and automatically respond to system events will only increase. Someday somebody will come up with a solution that works and that can be adopted with minimal pain; until then, `printk()` remains the only show in town.
