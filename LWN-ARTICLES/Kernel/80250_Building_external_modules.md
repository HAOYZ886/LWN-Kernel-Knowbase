---
title: Building external modules
url: https://lwn.net/Articles/80250/
date: "April 13, 2004"
category: Build system; Modules
author: 
---

Changes in the kernel build process have yielded a number of benefits in 2.6. They have, however, exposed a few rough edges for people building external modules. The [required procedure](<https://lwn.net/Articles/21823/>) is a bit inelegant, forces the user to ignore warnings from the build code ("you messed with SUBDIRS, do not complain if something goes wrong"), and does not support modversions. It also requires the presence of a configured and built kernel source tree, something which was not necessary with previous kernels, and a build of an external module will often try to rebuild things in the main tree as well. Fixing up the external module build process has been on the "to do" list for some time. 

Finally, somebody has done it. Sam Ravnborg has posted [a patch](<https://lwn.net/Articles/79984/>) which improves the external module build process in a number of ways. 

The basic form of a makefile for an external module will not change much. It should still look something like: 

```
ifneq ($(KERNELRELEASE),)
        obj-m	:= module.o
    
        else
        KDIR	:= /lib/modules/$(shell uname -r)/build
        PWD		:= $(shell pwd)
    
        default:
    	$(MAKE) -C $(KDIR) _M=$(PWD)_ 
        endif
```

The change has been underlined above; the parameter that once read `SUBDIRS=$(PWD)` has changed to `M=$(PWD)`. The older `SUBDIRS=` format will still work, however. It is also no longer necessary to specify the `modules` target when invoking the kernel build system. 

When the kernel build system is invoked with the `M=` parameter, it does a number of things differently. It will make no effort to ensure that the built files in the kernel source tree are current; if a developer makes a change to the main tree, it is his or her responsibility to rebuild it before trying to make any external modules. Only a few targets (`modules`, `clean`, `modules_install`) are supported when building external modules. And the `modpost` program now maintains a file (`Module.symvers`) containing the symbol version information if modversions is in use; this file is used when postprocessing an external module to note the symbol versions expected by that module. 

Among other things, the new scheme will allow distributors to package sufficient information for the building of external modules without the user having to actually configure and build the full kernel source tree. That information can be stored under `/lib/modules` by replacing the `build` symbolic link (which currently points back to the source tree) with a directory containing just the required information. That should make life simpler for everybody involved.
