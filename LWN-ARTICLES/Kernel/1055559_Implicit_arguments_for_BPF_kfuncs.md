---
title: Implicit arguments for BPF kfuncs
url: https://lwn.net/Articles/1055559/
date: "January 27, 2026"
category: "BPF-kfuncs"
author: "By Jonathan Corbet January 27, 2026"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 27, 2026

The kernel's "kfunc" mechanism is a way of exporting kernel functions so that they can be called directly from BPF programs. There are over 300 kfuncs in current kernels, ranging in functionality from string processing ([`bpf_strnlen()`](<https://elixir.bootlin.com/linux/v6.18.6/source/kernel/bpf/helpers.c#L3550>)) to custom schedulers ([`scx_bpf_kick_cpu()`](<https://elixir.bootlin.com/linux/v6.18.6/source/kernel/sched/ext.c#L6024>)) and beyond. Sometimes these kfuncs need access to context information that is not directly available to BPF programs, and which thus cannot be passed in as arguments. The [implicit arguments patch set](<https://lwn.net/ml/all/20260116201700.864797-1-ihor.solodrai@linux.dev>) from Ihor Solodrai is the latest attempt to solve this problem. 

The BPF subsystem maintains a collection of context information in the massive [`bpf_prog_aux` structure](<https://elixir.bootlin.com/linux/v6.18.6/source/include/linux/bpf.h#L1602>). Over the years it has grown to contain just about any sort of information that may be needed, including a list of maps the program is using, how long since it was loaded, the current stack depth, and dozens of other fields. It can almost be thought of as an analog to the `self` object passed to methods in some object-oriented languages. A number of kfuncs need to make use of this structure, but it is not something that BPF programs themselves can access. 

The way this problem has been handled in the BPF subsystem so far — the [`__prog` annotation](<https://docs.kernel.org/6.18/bpf/kfuncs.html#prog-annotation>) — cannot be said to be overly elegant. Consider [`bpf_stream_vprintk()`](<https://elixir.bootlin.com/linux/v6.18.6/source/tools/lib/bpf/bpf_helpers.h#L321>), which is defined as a macro: 

```
#define bpf_stream_printk(stream_id, fmt, args...)		\
        ({								\
    	static const char ___fmt[] = fmt;			\
    	unsigned long long ___param[___bpf_narg(args)];		\
    								\
    	___bpf_fill(___param, args);				\
    	bpf_stream_vprintk_impl(stream_id, ___fmt, ___param,	\
    		sizeof(___param), NULL);			\
        })
```

This macro marshals together the passed-in `args` and, in turn, calls [`bpf_stream_vprintk_impl`](<https://elixir.bootlin.com/linux/v6.18.6/source/kernel/bpf/stream.c#L354>), defined as: 

```
__bpf_kfunc int bpf_stream_vprintk_impl(int stream_id, const char *fmt__str, 
        					    const void *args, u32 len__sz,
    					    void *aux__prog);
```

The BPF verifier sees the trailing `__prog` in the name of the final parameter and magically replaces whatever was passed (the `NULL` in the macro above) with a pointer to the `bpf_prog_aux` structure. It gets the job done, but what is going on here is not exactly obvious to the casual observer. 

Solodrai first [attempted to generalize this mechanism](<https://lwn.net/Articles/1044824/>) in late 2025 with a "magic function" abstraction. In this implementation, the `__prog` suffix was switched to `__magic`, and the in-kernel version of the API (the one with the `__impl` suffix) was generated automatically. The concept was well received, but there were a lot of comments on the implementation; as a result, the patches have evolved somewhat since then. 

As the work stands now, a kfunc needing implicit arguments is declared normally, as it is expected to be invoked within the kernel. So, with the patch set applied, `bpf_stream_vprintk()` is declared as: 

```
__bpf_kfunc int bpf_stream_vprintk(int stream_id, const char *fmt__str, 
    				       const void *args, u32 len__sz,
    				       struct bpf_prog_aux *aux);
```

The pointer to the `bpf_prog_aux` structure is now without any magic adornments. Instead, when it comes time to inform the BPF subsystem about this kfunc, the declaration looks like: 

```
BTF_ID_FLAGS(func, bpf_stream_vprintk, KF_IMPLICIT_ARGS)
```

The `KF_IMPLICIT_ARGS` flag indicates that this function has an implicit argument that is not visible to BPF programs. 

When the kernel loads a BPF program containing a call to one or more kfuncs, the verifier looks up those kfuncs in the [BPF type format (BTF)](<https://docs.kernel.org/bpf/btf.html>) data created when the kernel is built. That data describes the kfunc and lets the verifier ensure that the BPF program is calling it correctly. The patch series causes the kernel-build process to modify the generated BTF for `KF_IMPLICIT_ARGS` kfuncs in a couple of ways: 

  * An entry for a new function, using the name of the kfunc with `_impl` appended, is added to the BTF data. This entry has the prototype for the kfunc as declared, and links to that function within the kernel. 
  * The entry for the kfunc using the original name (without "`__impl`") is modified so that the last argument is omitted. That is the BTF data that is used to match kfuncs for BPF programs. 

Whenever a call to a kfunc with implicit arguments is processed by the verifier, it modifies the call to add the `bpf_prog_aux` structure pointer as the final argument, essentially mapping the second form of the call above onto the first. So, in the end, the BPF program can access the kfunc with the expected API, with no need for wrapper macros to hide the work being done behind the scenes, and the kernel function can be declared under the name by which it will actually be used. 

`KF_IMPLICIT_ARGS` is plural, implying that there could be more than one argument but, for the time being, the only thing that works is a `struct bpf_prog_aux` pointer as the final argument. Should that eventually prove to be insufficient, Solodrai said, ""it is simple to extend this to more types"". Given the apparent practice of shoving everything that might be needed into `struct bpf_prog_aux`, though, there may never be a need for anything else. 

In the end, one could argue that the `bpf_prog_aux` pointer should have been implicitly passed to all kfuncs from the beginning. As it stands, this feature will make it easily and transparently available to the kfuncs that truly need it. There are already other patches (such as the [sched_ext sub-scheduler series](<https://lwn.net/ml/all/20260121231140.832332-1-tj@kernel.org>)) that depend on this feature, so it seems likely to land in the mainline in the relatively near future.
