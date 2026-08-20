---
title: MODULE_PARM deprecated
url: https://lwn.net/Articles/107303/
date: "October 20, 2004"
category: Development model; Modules
author: 
---

It's hard to turn down an opportunity to give Rusty Russell some grief, so let's take a moment to review [a comment he posted on LWN](<https://lwn.net/Articles/22768/>) in 2003: 

Regarding module_param(): MODULE_PARM() will certainly stay throughout the 2.6 series, so no need to change existing code just yet. 

Those who held off on changing their out-of-tree modules may want to do so now. Rusty has sent out [a patch marking `MODULE_PARM()` obsolete](<https://lwn.net/Articles/107304/>) in preparation for its removal from the kernel. A set of companion patches deals with many of the remaining `MODULE_PARM()` uses in the mainline tree. 

`MODULE_PARM()` declares parameters for loadable modules; these parameters can be changed when the module is loaded to affect its operation. One of the many changes that came with the new module loader in the 2.5 series was a new mechanism (`module_param()`) for declaring module parameters. The new scheme has a number of advantages over the old one: it is type safe, it allows module parameters to be represented (and changed) in sysfs, and it provides a flexible mechanism for new types of parameters. But, since the older way continued to work, many modules were never updated. 

Under the old development model, things probably would have gone as Rusty suggested: `MODULE_PARM()` would have remained through the 2.6 series in order to avoid breaking things. The new development model lacks the same sort of obvious demarcation point where compatibility can be broken, so those changes end up going into the regular patch stream. This is especially true of internal API changes, where there never has been a guarantee of any sort of continuity, even in an old-style stable series. So some of these changes are coming more quickly than some developers might have expected. 

With regard to `MODULE_PARM`, The current patches in circulation suggest that the time to update to `module_param()` is running out. Consider yourself warned.
