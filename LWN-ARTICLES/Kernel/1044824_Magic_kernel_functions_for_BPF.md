---
title: Magic kernel functions for BPF
url: https://lwn.net/Articles/1044824/
date: "November 10, 2025"
category: "BPF-kfuncs"
author: "By Daroc Alden November 10, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Daroc Alden**  
November 10, 2025

When programs written in BPF (the kernel's hot-loadable virtual-machine bytecode) call kernel functions (kfuncs), it may be useful for those functions to have additional information about the context in which those BPF programs are executing. Rather than requiring it to supply that information, it would be convenient to let the BPF verifier pass that information to the called function automatically. That is already possible, but [ a recent patch set](<https://lwn.net/ml/all/20251029190113.3323406-1-ihor.solodrai@linux.dev/>) from Ihor Solodrai would make it more ergonomic. It allows kernel developers to specify that a kfunc should be passed additional parameters inferred by the verifier, invisibly to the BPF program. The discussion included concerns that Solodrai's implementation was unnecessarily complex, however. 

Currently, kfuncs that need access to a BPF program's context information indicate this by adding the `__prog` annotation to one of their arguments. A BPF program calling the function passes `NULL` for that parameter, and the verifier automatically inserts a pointer to the program's [ `bpf_prog_aux` structure](<https://elixir.bootlin.com/linux/v6.17.7/source/include/linux/bpf.h#L1583>) (which can be used to find or manipulate a control group associated with a BPF program, for example). That is awkward because if a kfunc is updated to require a program's context information, all of the BPF programs that call it need to be updated to pass the extra parameter. 

Solodrai's [ first attempt](<https://lwn.net/ml/bpf/20250924211716.1287715-1-ihor.solodrai@linux.dev/>) at a fix for this was specific to functions that need the `bpf_prog_aux` structure. His most recent patch set adds a more general feature for "magic" function arguments of arbitrary types. 

> The usage of term "magic" is up for debate of course, I am open to suggestions. I used it as a placeholder first and now it weirdly makes sense. After all, "bpf" by itself doesn't mean anything either. 
> 
> [...] 
> 
> An accurate term could be something like "verifier provided arguments" and "kfuncs with verifier provided arguments", but that's too long for usage in the identifiers. "Magic" on the other hand is a short and ambiguous adjective, which hopefully will prompt people to check the documentation. 

The BPF verifier connects calls to kfuncs in a BPF program to the actual implementations of those functions using the kernel's BPF type format (BTF) debugging information. Magic functions are implemented by having two function signatures in the BTF: one for the kernel with all of the arguments (given a name ending in `_impl()`), and one for BPF programs, containing only the arguments that aren't provided by the verifier. BPF programs identify and call kfuncs using the BTF information, but the BPF verifier already needs to map the functions declared in BTF to actual addresses in the kernel and fix up argument types, so mapping two different BTF signatures to the same function is not a problem. BPF programs can technically call the function using either signature (i.e. preserving the current behavior of passing `NULL` for the magic parameters) for backward compatibility, but the magic version would be the preferred interface. 

In kernel code, a magic argument is indicated by adding `__magic` to the end of the argument name. This is similar to the way that other [ BPF argument annotations](<https://docs.kernel.org/bpf/kfuncs.html#annotating-kfunc-parameters>) work, but can be a bit non-obvious when first encountered. When pahole generates BTF during a kernel build, it recognizes the special format of the argument and treats it specially. For example: 

```
__bpf_kfunc int bpf_wq_set_callback(struct bpf_wq *wq,
            int (callback_fn)(void *map, int *key, void *value),
            unsigned int flags,
            struct bpf_prog_aux *aux__magic)
```

Eduard Zingerman [ suggested](<https://lwn.net/ml/all/b667472aeb77ac63a3de82dae77012c0285e0286.camel@gmail.com/>) "implicit" rather than "magic" as a name for these kinds of functions. Alexei Starovoitov [ agreed](<https://lwn.net/ml/all/CAADnVQJPevofLp0B0h40t0X1gj_032kw42K0ykHW4BqdmY0eNw@mail.gmail.com/>). Solodrai [ wasn't against](<https://lwn.net/ml/all/da20bc30-85be-44ab-b837-19aa97ebc431@linux.dev/>) changing the name, but worried about the confusion that could be caused by using names that already had associated meanings. 

Zingerman also [ wondered about](<https://lwn.net/ml/all/50ddff79d33d6e2d57e104f610273d647530ddbc.camel@gmail.com/>) how magic functions would impact compatibility with old versions of pahole. He later [ summarized](<https://lwn.net/ml/all/6e0b67d02cf7243b01a163589b3b58d1e4a0fdd8.camel@gmail.com/>) an off-list discussion about the problem that concluded that a slight change to pahole could minimize the compatibility problems. Specifically, when pahole sees a kfunc with an implicit argument, it would rename the function in the generated BTF and add a new function with the original name that doesn't take the implicit arguments; an old version of pahole would not know to do that, and would keep the name the same. Therefore, BPF programs built against a new kernel processed by an old pahole would not need to be adapted. 

The only scenario that would break is if a new kernel (using the new style of function declaration) were built using an old pahole, a BPF program was compiled against the produced `vmlinux.h`, and then that program were loaded into a kernel that was built with an updated pahole. This was deemed a sufficiently niche circumstance for a compatibility break. Alan Maguire [ asked](<https://lwn.net/ml/all/145364f5-3752-41b7-92d9-c97437b95b9a@oracle.com/>) why changes to pahole were needed, rather than trying to associate the two signatures for a magic function with each other inside the kernel. Zingerman [ explained](<https://lwn.net/ml/all/4558f35eb191c2605542b368f1dab4f69231ab8c.camel@gmail.com/>) that Andrii Nakryiko wanted to ensure that the kernel C code didn't need two separate declarations, and so pahole was the only place to add the second signature into the generated BTF. 

Nakryiko himself [ added](<https://lwn.net/ml/all/CAEf4Bza1cXRvw+v1CXmpWBF9ivnk+8-JWOUqRQJ2EE95j3i1Pw@mail.gmail.com/>) that it is important to consider two different kinds of compatibility at this point: the kernel's existing implicit functions (which have two declarations in the C code), and any new functions made with the new interface. The former don't have any problem for backward compatibility. The latter might, but even if there are plans to use them extensively in the future, they aren't being relied on by BPF programs today. In the future, the sched_ext extensible scheduler plans to make heavy use of this new feature. Nakryiko didn't explain why in his email, but it is likely related to the ongoing work that sched_ext has been doing to support hierarchically nested schedulers. 

With BPF's expanding role in the kernel, there is a lot to be said for making the process of exposing kernel features to BPF programs as easy as possible. Whether that justifies adding another contortion to the kernel's already somewhat convoluted BTF information remains to be seen. Solodrai's patch set seems likely to go through another revision before reaching its final form, but the BPF maintainers are clearly onboard with the core idea.
