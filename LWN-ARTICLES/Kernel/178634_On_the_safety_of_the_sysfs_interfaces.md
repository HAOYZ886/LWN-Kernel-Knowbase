---
title: On the safety of the sysfs interfaces
url: https://lwn.net/Articles/178634/
date: "April 5, 2006"
category: Security; Sysfs
author: 
---

One of the [patches](<https://lwn.net/Articles/178635/>) in the upcoming 2.6.16.2 stable kernel release is a fix for a security vulnerability designated as CVE-2006-1055. It makes a small change to the code which implements the ability to write to sysfs attributes; with this change, the maximum amount of data which can be written to an attribute is `PAGE_SIZE-1` bytes, or 4095 on most systems. Since last June, the limit had simply been `PAGE_SIZE`, allowing a full page to be written. 

Since the page is zeroed before being filled, this change ensures that the data coming from user space will be null-terminated when it is passed to the specific sysfs `store()` function. Without that assurance, that function might have proceeded merrily off the end of the one-page buffer, accessing data which did not come from user space and possibly overwriting buffers elsewhere. The possibility of this happening was enough to raise security fears and motivate a quick fix. 

The interesting thing is that the prototype for the `store()` function is: 

```
ssize_t (*store)(struct kobject *kobj, struct attribute *attr,
                         const char *buffer, size_t size);
```

The `size` parameter is the amount of user data being passed in. So, one might ask, why bother null-terminating the buffer, when its size has already been made available to the receiving code? Certain developers, whose code was receiving 4096-byte data via sysfs attributes, have, indeed, asked that question. 

The question was answered, in one way, in the message featured in the [quote of the week](<http://lwn.net/Articles/178630/>). More diplomatically, one might say that, regardless of how the interface was designed, a number of sysfs attribute implementations have be coded on the assumption that the incoming data will be null-terminated. So they do not bother to check the length of that data, and they will do bad things in the absence of the expected terminator. 

With the 2.6.16.2 patch, the situation will be fixed and those implementations made safe again. But it is hard not to be a little nervous about the situation. If there is carelessly-written code in the tree, there may be other issues with it as well, and the return of null-termination may not help much. It would be nicer if there were a way to verify that the interfaces were being used correctly. In the mean time, people writing sysfs interfaces - each of which is an interface to user space and a possible target of attack - may want to look a little more carefully at their code before submitting it.
