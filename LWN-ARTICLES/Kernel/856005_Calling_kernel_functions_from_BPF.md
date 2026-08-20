---
title: Calling kernel functions from BPF
url: https://lwn.net/Articles/856005/
date: "May 13, 2021"
category: "BPF-kfuncs; Releases-5.13"
author: "By Jonathan Corbet May 13, 2021"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 13, 2021

The kernel's [BPF](<https://lwn.net/Articles/740157/>) virtual machine allows programs loaded from user space to be safely run in the kernel's context. That functionality would be of limited use, however, without the ability for those programs to interact with the rest of the kernel. The interface between BPF and the kernel has been kept narrow for a number of good reasons, including safety and keeping the kernel in control of the system. The 5.13 kernel, though, contains a feature that could, over time, widen that interface considerably: the ability to directly call kernel functions from BPF programs. 

The immediate driver for this functionality is the implementation of TCP congestion-control algorithms in BPF, a capability that was [added to the 5.6 kernel release](<https://lwn.net/Articles/811631/>) by Martin KaFai Lau. Actual congestion-control implementations in BPF turned out to reimplement a number of functions that already exist in the kernel, which seems less than fully optimal; it would be better to just use the existing functions in the kernel if possible. The new function-calling mechanism — also implemented by Lau — makes that possible. 

#### Making functions available to BPF

On the BPF side, using a kernel function is now just a matter of declaring it `extern` and calling it like any other C function. Within the kernel, instead, a bit more work has to be done. BPF programs are meant to only have access to a specific set of allowed functions, and that set is only available to the intended BPF program type; code inside the kernel must thus make those functions available in the right context. So, for example, [this commit](<https://git.kernel.org/linus/e78aea8b2170>) makes [`tcp_slow_start()`](<https://elixir.bootlin.com/linux/v5.13-rc1/source/net/ipv4/tcp_cong.c#L386>) available to BPF — but only for congestion-control programs. 

"Exporting" functions to BPF programs is done by adding a new function to the [`bpf_verifier_ops` structure](<https://elixir.bootlin.com/linux/v5.13-rc1/source/include/linux/bpf.h#L461>) associated with the program type: 

```
bool (*check_kfunc_call)(u32 kfunc_btf_id);
```

This function will be called by the BPF verifier when it encounters an external call; `kfunc_btf_id` is the [BPF type format (BTF)](<https://www.kernel.org/doc/html/latest/bpf/btf.html>) ID assigned to the function that the BPF program wants to call. The function should return true if the call should be allowed. If `tcp_slow_start()` were the only function to be made available in this way, that function could be written as: 

```
static bool bpf_tcp_ca_check_kfunc_call(u32 id)
        {
        	return id == BTF_ID(func, tcp_slow_start);
        }
```

If there are many functions to export, there are easier ways than a long list of `if` statements to do the checking; see the above-linked commit for an example. 

Beyond checking that the function is meant to be available, the BPF verifier carries out a number of other checks. For example, the arguments passed to the function and their types must be correct, or the program will be rejected. The call is only allowed if the verifier can convince itself that it is safe, though the verifier obviously cannot really know what is going on inside the called function or the ways in which things could go wrong. 

#### Some questions

So far, congestion-control programs are the only program type to make use of this feature, but it is not hard to imagine that others will come in the future. There are a number of interesting questions that are raised by this capability and how it might be used going forward. 

The first of those might be: how does this capability differ from the BPF helper mechanism that has been part of BPF for years? The changelog does not address that question, so your editor has to guess. BPF helper programs must be written explicitly for use from BPF programs, must be declared specially, and require a [`bpf_func_proto` structure](<https://elixir.bootlin.com/linux/v5.13-rc1/source/include/linux/bpf.h#L333>) to be filled in and made available to the verifier; see [the setup for `bpf_map_lookup_elem()`](<https://elixir.bootlin.com/linux/v5.13-rc1/source/kernel/bpf/helpers.c#L20>) for an example. Making an existing kernel function available as a BPF helper means writing a wrapper function, then going through this whole dance. 

To make a kernel function callable, instead, is just a matter of defining a "check" function that allows the call to happen, and the BPF subsystem does the rest. One could argue that helpers should have been implemented that way in the first place, but there is a lot of necessary infrastructure that only showed up years after the helper mechanism was developed. Without BTF, this would not be possible; the BPF Linux security module (formerly [KRSI](<https://lwn.net/Articles/808048/>)) also brought some of the necessary support. Had that infrastructure existed at the beginning, it's possible that there would never have been a need to add BPF helpers. 

That said, BPF helpers have the advantage of existing solely for use by BPF programs; kernel functions are there to be called by the rest of the kernel. There is no stable ABI within the kernel, so it would not be surprising to see the interface to BPF-exported kernel functions change more often than the interface to BPF helpers. The [commit adding the function-calling capability](<https://git.kernel.org/linus/e6ac2450d6de>) makes a clear statement that there are no ABI guarantees: 

> The white listed functions are not bounded to a fixed ABI contract. Those functions have already been used by the existing kernel tcp-cc. If any of them has changed, both in-tree and out-of-tree kernel tcp-cc implementations have to be changed. 

It will be interesting to see what happens if an internal kernel change breaks a high-profile BPF program and users start to complain. It is generally understood that functionality provided to BPF is not part of the kernel ABI, but that policy has never been explicitly blessed by Linus Torvalds or seriously tested. 

BPF helpers are also designed to be safely called from the BPF context — from outside of the kernel itself, in other words. Regular kernel functions are not written with a possibly hostile caller in mind. The BPF subsystem as a whole goes to great lengths to ensure that a BPF program cannot crash or compromise the system, but that subsystem cannot know what happens inside some kernel function and cannot guarantee that the arguments to any given function call make sense. If the wrong functions are made available to BPF, an erroneous or hostile program could use them to make an unpleasant mess. 

Finally, this mechanism looks a bit like a backdoor way to export kernel symbols outside of the kernel itself. The exporting of symbols to modules requires an `EXPORT_SYMBOL()` declaration next to the relevant code and often attracts a fair amount of attention and debate over whether kernel internals should be exposed in that way. Exporting of functions to BPF programs is a lower-profile activity that can happen far away from the definition of the functions involved. In an extreme case, there does not appear to be anything to prevent somebody from registering a checking function like this: 

```
static bool export_the_world(u32 kfunc_btf_id)
        {
        	return true;
        }
```

The result of adding this function would be to make almost any kernel function callable from a BPF program of the right type. That is unlikely to be seen as a good outcome. In theory such a function would be caught in review, but it is worth asking how many people have reviewed the [test code](<https://elixir.bootlin.com/linux/v5.13-rc1/source/net/bpf/test_run.c#L218>) for function calls from BPF that has been added (as part of this patch series) to the (entirely unrelated) traffic-control classifier program type; this (harmless) code will be present in all systems with traffic control enabled. It does not seem that it would be hard to add a severe bug, intentionally or otherwise, by exporting the wrong function to BPF programs. 

Some of these concerns could perhaps be mitigated by registering a list of allowed kernel functions with the BPF core rather than supplying a function that makes its own decisions. That is not how this feature was implemented, though. 

Be that as it may, the BPF function-calling mechanism has been merged and will be included in the 5.13 release. Presumably there will be enough vigilance to keep kernel functions from being inappropriately exported in the mainline kernel in future releases. Properly managed, this feature could be used to make a great deal of functionality available to BPF programs, significantly growing the set of useful things that can be done with BPF. It will be interesting to see where this feature goes from here.
