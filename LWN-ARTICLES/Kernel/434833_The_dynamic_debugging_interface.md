---
title: The dynamic debugging interface
url: https://lwn.net/Articles/434833/
date: "March 22, 2011"
category: Debugging; Development tools
author: "By Jonathan Corbet March 22, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 22, 2011

The kernel's "dynamic debugging" interface saw some minor changes for 2.6.39. As it happens, LWN has never written about how dynamic debug works, so this seems like an opportune time to fill in the gap. 

It can be nice to instrument kernel code with abundant print statements that illustrate what is going on inside. The problem, of course, is that those statements can generate vast amounts of output which is usually not of interest. These statements can be left commented out most of the time, but that leads to situations where an edit/rebuild/reboot cycle is needed to get the output. In response, many developers have created mechanisms which enable or disable specific print statements at run time. The dynamic debugging interface was added as a way of providing a uniform control interface for debugging output while avoiding cluttering the kernel with various hand-rolled alternatives. 

Dynamic debug operates on print statements written with either of: 

```
pr_debug(char *format, ...);
        dev_dbg(struct device *dev, char *format, ...);
```

If the `CONFIG_DYNAMIC_DEBUG` option is not set, the above functions will be turned into normal `printk()` statements at the `KERN_DEBUG` level. If the option is enabled, though, the code sets aside a special descriptor for every call site, noting the module, function, and file names, along with the line number and format string. At system boot, all of these debug statements are turned off, so their output will not appear even if debug-level kernel messages are routed somewhere useful by the syslog daemon. 

Turning on dynamic debug causes a new virtual file to appear at `/sys/kernel/debug/dynamic_debug/control` (modulo any individual preferences for the location of debugfs, naturally). Writing to that file will enable or disable specific debugging functions, as specified by a simple but flexible language. 

As an example, `drivers/char/tpm/tpm_nsc.c` contains the following code at line 346: 

```
dev_dbg(&pdev->dev, "NSC TPM detected\n");
```

Turning on that specific line could be done with a line like: 

```
echo file tpm_nsc.c line 346 +p > .../dynamic_debug/control
```

(Where the full path to debugfs has been replaced with "`...`"). As it happens, that `dev_dbg()` line does not stand alone - there is a long series of them providing information on the newly-detected device. One could enter a series of lines like the above to enable them all individually, but either of the following would also work: 

```
echo file tpm_nsc.c line 346-373 +p > .../dynamic_debug/control
        echo file tpm_nsc.c function init_nsc +p > .../dynamic_debug/control
```

Along with selection by file name, line number, and function name, the interface also allows "`module _name_` " to select a specific module, and "`format _fmt_` " to select any line whose format string contains "`_fmt_` ". If more than one selector is given, all must match for a given statement to be enabled. 

Commands to the control file must end with a "flags" operation telling the system what to do; "`+p`" turns on `printk()` output, while "`-p`" turns it off. There is also a set of flags (new for 2.6.39) controlling information added to each output line: "`f`" adds the function name, "`l`" adds the line number, "`m`" adds the module name, and "`t`" adds the thread ID. One can use "`=`" to set the full mask of flags to a specific value - "`=plm`" will enable printing with line numbers and module names while disabling thread ID and function output regardless of their prior setting. The only way to clear all of the flags is with "`-pflmt`". 

Reading the control file will produce a list of all currently-enabled call sites. 

Sometimes the interesting action happens before the system reaches a point where the control file can be accessed. Dynamic debug output can be turned on early in the boot process with the `ddebug_query` boot parameter. 

More information on how to use this facility can be found in [Documentation/dynamic-debug-howto.txt](<https://lwn.net/Articles/434856/>). Dynamic debug has been in the kernel since 2.6.30, but it is still common to see code submitted which contains its own, home-brewed mechanism for controlling debug output. Chances are that reviewers will ask for such mechanisms to be taken out before the code is merged. Given the flexibility and ease of use of the in-kernel implementation, it makes sense to use it from the beginning.
