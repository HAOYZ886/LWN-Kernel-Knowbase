---
title: A common clock framework
url: https://lwn.net/Articles/472998/
date: "December 21, 2011"
category: Clocks
author: "By Jake Edge December 21, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
December 21, 2011

One of the big problem areas that has been identified in the ARM kernel trees is the diversity of implementations for various things that could be shared--either within the ARM tree or more widely with the rest of the kernel. That problem has led to a large amount of duplicated code in the ARM tree, both via cut-and-paste and code that is conceptually similar but uses different data structures and APIs. The latter makes the creation of a single kernel image that can boot on multiple ARM platforms impossible, so there are efforts to consolidate these implementations. The common clock framework is one such effort. 

In a typical ARM system-on-chip (SoC), there can be dozens of different clocks for use by various I/O and other devices in the SoC. Typically those clocks are hooked together into elaborate tree-like structures. In those trees, child clocks can sometimes only change their frequency if the parent (and any other children) are correspondingly changed; disabling certain clocks will affect other clocks in the system and so on. Each ARM platform/SoC has its own way of encapsulating that information and presenting it to other parts of the system (like power and thermal management controllers), which makes it difficult to create platform-independent solutions. 

The first problem that a common clock framework faces is the sheer number of different `struct clk` definitions scattered throughout the ARM tree. There are more than two dozen definitions in `arch/arm` currently, but the [proposal](<https://lwn.net/Articles/472093/>) for a common framework not surprisingly reduces that number to one. Implementations can wrap the `struct clk` in another structure that holds hardware-specific data, but the common structure looks like: 

```
struct clk {
    	const char                  *name;
    	const struct clk_hw_ops     *ops;
    	struct clk                  *parent;
    	unsigned long               rate;
    	unsigned long               flags;
    	unsigned int                enable_count;
    	unsigned int                prepare_count;
    	struct hlist_head           children;
    	struct hlist_node           child_node;
        };
```

The `parent` and `children`/`child_node` fields allow the clocks to be arranged into trees, while the `rate` field tracks the current clock frequency (in Hz). The `flags` field is used to describe the clock type (e.g. whether a rate change needs to be done on the parent clock, or that the clock must be disabled before changing the rate). The two `*_count` fields are for tracking calls to the enable and prepare operations, while the bulk of the "work" is done within the `struct clk_hw_ops` field (`ops`). 

Each of the entries in the `clk_hw_ops` structure correspond to a function in the driver-facing API for the clock framework. That API does some sanity checking before calling the corresponding operation from `clk_hw_ops`: 

```
struct clk_hw_ops {
    	int             (*prepare)(struct clk *clk);
    	void            (*unprepare)(struct clk *clk);
    	int             (*enable)(struct clk *clk);
    	void            (*disable)(struct clk *clk);
    	unsigned long   (*recalc_rate)(struct clk *clk);
    	long            (*round_rate)(struct clk *clk, unsigned long,
    				      unsigned long *);
    	int             (*set_parent)(struct clk *clk, struct clk *);
    	struct clk *    (*get_parent)(struct clk *clk);
    	int             (*set_rate)(struct clk *clk, unsigned long);
        };
```

`clk_prepare()` is used to initialize the clock to a state where it could be enabled, and that call must be made before `clk_enable()`, which actually starts the clock running. `clk_disable()` and `clk_unprepare()` do the reverse and should be called in that order. The difference is that `clk_prepare()` can sleep, while `clk_enable()` must not, so having two separate calls allows the clock initialization to be split into atomic and non-atomic pieces. 

`clk_get_parent()` and `clk_set_parent()` do what the names imply, simply returning or changing the `parent` field, though setting the parent only succeeds if the clock is not already in use (otherwise -EBUSY is returned). `clk_recalc_rate()` queries the hardware, rather than the cached `rate` field, for the current frequency of the clock. `clk_round_rate()` rounds a frequency in Hz to a rate that the clock can actually use, and can also be used to determine the correct frequency for the parent clock when changing rates. All of those are more or less helper functions for `clk_set_rate()`. 

`clk_set_rate()` changes the frequency of a clock, but it must take into account some other factors. If the `CLK_PARENT_SET_RATE` flag value is set for the clock, `clk_set_rate()` needs to propagate the change to the parent clock (which may also have that flag set, necessitating a recursive traversal of the tree, attempting to set the rate at each level). 

Drivers can also register their interest in being notified of rate changes with the `clk_notifier_register()` function. Three different types of notification can be requested: before the clock's rate changes, after it has been changed, or if the change gets aborted after the pre-change notifications have been called (i.e. `PRE_RATE_CHANGE`, `POST_RATE_CHANGE`, and `ABORT_RATE_CHANGE`). In each case, both the old and new values for the rate get passed as part of the notification callback. The [patch](<https://lwn.net/Articles/473214/>) to add notifications creates another operation in `clk_hw_ops` called `speculate_rate()`, which notes potential rate changes and sends any needed pre-change notifications as it walks the sub-tree. 

The patch set also exports the clock hierarchy into debugfs. Each top-level clock gets a directory in `../debug/clk` that contains read-only files to report the clock's rate, flags, prepare and enable counts, and the number of notifiers registered. Subdirectories are created for each child clock containing the same information. 

The common clock framework has been around for some time in various forms. The current incarnation is being shepherded by Mike Turquette, but he [notes](<http://article.gmane.org/gmane.linux.kernel/1218622>) that it is based on work originally done by Jeremy Kerr and Ben Herrenschmidt. Beyond that: ""Many others contributed to those patches and promptly had their work stolen by me"". Turquette has also [posted](<https://lwn.net/Articles/473209/>) a patch set with an example that replaces the OMAP4 clocks using the framework. 

The comments on the most recent iteration have been fairly light, but still substantive, so we are clearly a ways off from seeing a version in the mainline. It's clearly on the radar of ARM developers, and would clean up a fair amount of code duplication within that tree, so we should see something in the mainline soon--hopefully in one of the next few kernel releases.
