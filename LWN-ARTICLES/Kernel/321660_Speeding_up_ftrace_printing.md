---
title: Speeding up ftrace printing
url: https://lwn.net/Articles/321660/
date: "March 4, 2009"
category: Ftrace
author: "By Jake Edge March 4, 2009"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
March 4, 2009

A kernel patch that reduces memory, while providing a performance increase of roughly a factor of three, is generally seen as a good thing. But, when there is another, more-or-less equivalent--but much faster--way to perform that action, it may appear to be an unnecessary optimization. A recent [patch to the `ftrace_printk()` function](<https://lwn.net/Articles/321730/>) has those characteristics, but the ability to get such a speed increase, even in something that is just convenient--rather than required--may well trump the concerns about the necessity. 

Lai Jiangshan proposed adding a binary version of `ftrace_printk()` last December; Frederic Weisbecker has picked up the patches and submitted them for inclusion into ftrace. The basic idea is that rather than converting the arguments to strings--as specified in a `printk()`-style format string--`ftrace_bprintk()` would defer the actual conversion until the trace output is read by user space. Instead it would put the binary values into the ring buffer, along with a pointer to the format string. When the trace data is read from `debugfs`, the format string and binary data are used to construct the output. 

Ingo Molnar [liked the idea](<https://lwn.net/Articles/321742/>), but was unhappy with the implementation that duplicated much of the code in `vsnprintf()` into two new functions. He suggested that it should be possible to pull out the common code: ""We should try _much_ harder at unifying these functions before giving up and duplicating them."" Weisbecker agreed, which eventually resulted in a [patch](<https://lwn.net/Articles/321739/>) that breaks out the format string decoding as a separate function. 

Molnar also asked for some performance numbers, which Weisbecker provided as part of his patch. He reported the memory and time difference when adding: 

```
ftrace_printk("This is the timer interrupt: %llu", jiffies_64);
```

to the timer interrupt. The memory used was less than half (16 versus 39 bytes per entry), and the time savings was also significant: 

After some time running on low load (no X, no really active processes): 

```
ftrace_printk:  duration average: 2044 ns, avg of bytes stored per entry: 39
        ftrace_bprintk: duration average: 1426 ns, avg of bytes stored per entry: 16
```

Higher load (started X and launched a cat running on an X console looping on traces printing): 

```
ftrace_printk:  duration average: 8812 ns
        ftrace_bprintk: duration average: 2611 ns
```

Andrew Morton was [a bit puzzled](<https://lwn.net/Articles/321743/>) by the intent of the patch: ""Trying to make something which is inherently slow run slightly faster seems...odd."" But Molnar [explained](<https://lwn.net/Articles/321772/>) why it makes sense: 

The _fastest_ way of tracing is obviously to know about the precise argument layout and having a specific C based tracepoint stub that directly stuffs that data into the ring buffer. Most tracepoints are of such nature. 

That does not remove the ease of use of ad-hoc printk-alike tracepoints though, and speeding them up 3-fold is a [worthwhile] goal. 

Breaking out the format string handling into its own `format_decode()` function was mostly met with approval, except that the argument list is rather ugly: 

```
int format_decode(const char *fmt, enum format_type *type, 
                          int *flags, int *field_width, int *base, 
                          int *precision, int *qualifier)
```

Linus Torvalds [suggested](<https://lwn.net/Articles/321773/>) using a `struct printf_spec` to contain the various values decoded from the format specifier, passing a pointer to that into the function. Weisbecker agreed, and added that into his patches, but he didn't quite go far enough. 

Torvalds also thought that the various helper functions to handle specific formats (i.e. `number()`, `pointer()`, `string()`, etc.) should get passed a `struct printf_spec` pointer as well. As he [points out](<https://lwn.net/Articles/321774/>): ""When cleaning up, let's just do it properly."" Once again, Weisbecker was quick to agree; he plans to respin the patches addressing these and other comments in the near future. 

In addition, because `ftrace_bprintk()` is a drop-in replacement for `ftrace_printk()`, Weisbecker [proposes](<https://lwn.net/Articles/321779/>) eliminating the current code in favor of the faster version. Molnar, at least, [advocates](<https://lwn.net/Articles/321780/>) that outcome: 

Well, ftrace_bprintk() seems to be a worthy and transparent replacement for ftrace_printk() to me. I.e. lets just use this as the new implementation for ftrace_printk(). 

While it is a minor upgrade to a relatively minor kernel subsystem, it does provide some impressive performance gains. As a bonus, the review process has resulted in some clean-up that was probably overdue. While there is validity to the argument that it is not really _required_ , it is not very intrusive, nor very large. In the end, that is likely to be enough to see it eventually end up in the mainline.
