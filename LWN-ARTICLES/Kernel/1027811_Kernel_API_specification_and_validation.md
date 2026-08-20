---
title: Kernel API specification and validation
url: https://lwn.net/Articles/1027811/
date: "July 3, 2025"
category: "Development tools-ABI-related"
author: "By Jonathan Corbet July 3, 2025"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 3, 2025

The kernel project makes a strong promise to its users: the kernel ABI will not be changed in ways that break user-space code. The occasional failure notwithstanding, kernel developers do try to live up to that promise. They are handicapped by one little problem, though: there is no description of what the kernel ABI is, and no comprehensive way to test whether a given change breaks it. The [kernel API specification framework](<https://lwn.net/ml/all/20250624180742.5795-1-sashal@kernel.org>) proposed (in its second revision) by Sasha Levin addresses some of those concerns, but the solution is incomplete and does not come for free. 

(Note that Levin uses the term "API" rather than "ABI" throughout this work; that term will be used from here on as well.) 

The kernel interface is complex. It includes hundreds of system calls, many of which have complex parameters and behavior; all of that must be completely described if a specification is to be complete. There are other aspects to the API as well, though, including files in `/proc` or `/sys`, memory-mapped regions created by the perf-events subsystem or io_uring, and the set of operations available for any given type of file descriptor. Even more complexity comes with the interfaces available to BPF programs or loadable kernel modules, though those are not covered by the kernel's API guarantee. Levin's patch set does not cover all of those areas, but it makes a good start. 

#### Specifying system calls

Much of the framework, as posted, is focused on defining system calls. The actual specification for any given system call is formatted as a long set of macro calls within the source file for the system call itself. Consider, for example, [the specification](<https://lwn.net/ml/all/20250624180742.5795-13-sashal@kernel.org>) for the (relatively simple) [`mlockall()`](<https://man7.org/linux/man-pages/man2/mlock.2.html>) system call; it contains a large amount of text in a block like this: 

```
DEFINE_KERNEL_API_SPEC(sys_mlockall)
    	KAPI_DESCRIPTION("Lock all process pages in memory")
    	KAPI_LONG_DESC("Locks all pages mapped into the process address space. "
    		       "MCL_CURRENT locks current pages, MCL_FUTURE locks future mappings, "
    		       "MCL_ONFAULT defers locking until page fault.")
    	/* ... lots more ... */
        KAPI_END_SPEC;
```

The `DEFINE_KERNEL_API_SPEC()` call begins the definition of a structure of type `kernel_api_spec`; `KAPI_END_SPEC`, defined simply as "`};`", ends that structure. The macro calls in between fill in the fields of that structure in various ways. `KAPI_DESCRIPTION()` for example, is defined simply as: 

```
#define KAPI_DESCRIPTION(desc) \
    	.description = desc,
```

There is only one parameter to `mlockall()` — a flags value. It is specified as: 

```
KAPI_PARAM(0, "flags", "int", "Flags controlling which pages to lock")
    		KAPI_PARAM_FLAGS(KAPI_PARAM_IN)
    		KAPI_PARAM_TYPE(KAPI_TYPE_INT)
    		KAPI_PARAM_CONSTRAINT_TYPE(KAPI_CONSTRAINT_MASK)
    		.valid_mask = MCL_CURRENT | MCL_FUTURE | MCL_ONFAULT,
    		KAPI_PARAM_CONSTRAINT("Must specify MCL_CURRENT and/or" \
    				MCL_FUTURE; MCL_ONFAULT can be OR'd")
        KAPI_PARAM_END
```

`KAPI_PARAM()` begins the initialization of a `kapi_param_spec` structure nested within the `kernel_api_spec` structure. Here, that declaration says that the first parameter (number 0) is an `int` value called `flags`; it is an input parameter to the system call. There are three valid flag bits, which are stored directly in `valid_mask` without a special macro call. An extra constraint on those values (at least one of `MCL_CURRENT` and `MCL_FUTURE` must be set) is just provided as text; there is not a way to specify that sort of constraint in the current system. 

That is a fair amount of upper-case text, but this specification is just beginning. It also contains declarations describing the return value and the error codes that the system call can return. Other declarations describe how the system call responds to various signals. There are six blocks of declarations describing the system call's side effects, and five for "state transitions". The internal kernel locks used by the system call are described. There is a declaration block for the capabilities required by `mlockall()`, one for examples, and three additional constraints. All told, the specification for `mlockall()` is just over 180 lines. 

The [specification for `socket()`](<https://lwn.net/ml/all/20250624180742.5795-22-sashal@kernel.org>) is, as one might expect, rather longer. 

#### Sysfs and more

The second version of the patch set added the ability to specify the behavior of sysfs attributes as well. A simple example from the block layer looks like this: 

```
DEFINE_SYSFS_API_SPEC(partscan)
    	KAPI_DESCRIPTION("Partition scanning status")
    	KAPI_LONG_DESC("Reports if partition scanning is enabled for the disk. "
    		       "Returns '1' if partition scanning is enabled, or '0' if not.")
    	KAPI_PARAM_COUNT(1)
    	KAPI_PARAM(0, "partscan", "bool", "Partition scanning enabled flag")
    		KAPI_PARAM_TYPE(KAPI_TYPE_BOOL)
    		KAPI_PERMISSIONS(0444)
    		KAPI_PATH("/sys/block/<disk>/partscan")
    		KAPI_PARAM_FLAGS(KAPI_PARAM_SYSFS_READONLY)
    	KAPI_PARAM_END
    	KAPI_SUBSYSTEM("block")
    	KAPI_EXAMPLES("cat /sys/block/sda/partscan")
    	KAPI_NOTES("The value type is a 32-bit unsigned integer, "\
    		   "but only '0' and '1' are valid values")
        KAPI_END_SPEC;
```

There is less that can be specified with regard to a sysfs value, so these descriptions tend to be somewhat shorter. 

As is almost always the case, [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) is special, since every `ioctl()` call is different. There is a set of macros for the specification of these calls; see [the specification for the fwctl subsystem](<https://lwn.net/ml/all/20250624180742.5795-18-sashal@kernel.org>) for an example. There is also a format for the specification of internal kernel functions. 

#### Then what?

The ability to precisely describe a kernel interface has value in its own right; it forms a sort of documentation (there is no integration with the kernel's existing documentation system, though that is evidently on the list for future work). But documentation alone seems unlikely to be enough to inspire kernel developers to write and maintain that much additional text, or even to accept it into their code if somebody else does it. The real value comes from what can be done with all of this specification data once it has been created. 

If the kernel is configured for it, all of the specification data will be built into the kernel image. That adds up to 4KB of memory for each API specification, with the result that a fully specified kernel may contain several megabytes of added data. So this is probably not an option to enable in production kernels, but all of that data can be useful in development and testing scenarios. 

The specification data is made available under debugfs; a simple `cat` command can thus be used to extract a human-readable rendering of any given specification; JSON and XML renderings are available as well. There are special `all.json` and `all.xml` debugfs files that can be used to extract the entire set of specification data in JSON or XML format. 

Kernels with this data built in can be set up to validate system calls (and internal functions) against their specifications. Most of this validation, at the moment, consists of ensuring that the parameters passed into the function comply with the specification; they are thus, in a sense, validating the caller, not the function itself. There is also checking of a function's return value, with the ability to fail a system call entirely if it returns something unexpected. There are functions to validate the lock specification and signal handling, but they are stubs in the current implementation. Developers can also supply custom validation functions for a specific function. 

Finally, there is a new tool called `kapi`, written ([mostly automatically](<https://lwn.net/ml/all/aGP2rMDoJGd9fB4s@lappy>)) in Rust, that can be used to obtain specification data. It is able to read that data from the source directly, from a built kernel image, or from debugfs. The output is available as plain text, JSON, or ReStructuredText. Among other things, `kapi` can be used to check for API differences between two different kernels. 

One of the key goals of this series is to enable the automatic detection of API-breaking changes. In its current state, though, it appears that it will mostly detect changes where the developer was diligent enough to update the specification to reflect a change elsewhere, in which case the break is already known to that developer. The validation can catch some changes, but will surely miss many others. 

It is necessary to start somewhere, though. In time, if this framework shows enough potential, it could perhaps grow into something that can cover much of the kernel API and ensure its consistency. Future plans include integration with static analysis to verify the API specification, integration with fuzzing tools for smarter testing, and low-overhead run-time validation that can be enabled in production kernels. But, to get there, Levin will have to convince the development community that the existing framework provides enough value to justify the addition and maintenance of thousands of lines of specification data. That conversation is just beginning.
