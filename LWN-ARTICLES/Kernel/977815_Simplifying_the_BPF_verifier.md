---
title: Simplifying the BPF verifier
url: https://lwn.net/Articles/977815/
date: "June 13, 2024"
category: "BPF-Verifier"
author: "By Daroc Alden June 13, 2024 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Daroc Alden**  
June 13, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The BPF verifier is a complex program. This has the unfortunate effect of making it simultaneously more difficult for contributors to work on, and more likely to harbor unknown bugs. Shung-Hsi Yu had two concrete proposals for how to simplify the verifier to make it easier to maintain that he presented at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). Yu proposed changing how the verifier tracks partially known values and cleaning up the interface to hide the details of the value-tracker's internal representation. 

[ ![\[Shung-Hsi Yu\]](https://static.lwn.net/images/2024/shung-hsi-yu-small.png) ](<https://lwn.net/Articles/977990>)

One of the core functions of the verifier is value tracking — inferring the set of possible values that a variable can hold, in order to ensure that accesses remain within bounds. Since any value could potentially be used to compute an array index or other quantity the verifier is interested in knowing, the value tracker needs to follow every value in the program. The verifier stores information on possible values in the `bpf_reg_state` structure, which tracks two related kinds of information. The first is "known bits", which uses a mask to indicate when individual bits of the value are known exactly: 

```
struct tnum {
            u64 value;
            u64 mask;
        }
```

The second is the valid range of the value, tracked as both signed and unsigned 32- and 64-bit quantities: 

```
struct bpf_reg_state {
            ...
            struct tnum var_off;
            s64 smin_value; /* minimum possible (s64)value */
            s64 smax_value; /* maximum possible (s64)value */
            u64 umin_value; /* minimum possible (u64)value */
            u64 umax_value; /* maximum possible (u64)value */
            s32 s32_min_value; /* minimum possible (s32)value */
            s32 s32_max_value; /* maximum possible (s32)value */
            u32 u32_min_value; /* minimum possible (u32)value */
            u32 u32_max_value; /* maximum possible (u32)value */
        }
```

This choice of what information to track represents a tradeoff between accuracy and efficiency. If computers had overabundant memory, the verifier could just track the set of possible values directly using a generic set data structure. The downside of that approach would be the significantly increased memory overhead compared to the bytes required to store a `bpf_reg_state`. The downside of the more efficient approach is that it can't represent all possible sets of values, so sometimes the code needs to make a conservative over-approximation, which can snowball and make the verifier fail to figure out bounds that it theoretically could have. For example, the verifier can't currently handle a disjoint range, like a value that must be between one and four _or_ eight and ten. Instead, it would track the range as just one to ten. 

In practice, tracking both known bits and possible ranges provides a good tradeoff. Either one alone would fail to capture important properties that the verifier cares about, but together they aren't too large or complex to work with. They can represent possible sets of values such as "a multiple of eight between zero and 64", which is a good fit for tracking the alignment and bounds of an array access. 

#### Track fewer bounds

Yu has a proposal that could simplify the actual implementation of `bpf_reg_state` significantly, while still keeping the same precision: stop tracking the signed versions of the ranges separately. Right now, whenever the verifier updates one range (such as inferring a new `smin_value` from a conditional branch), it needs to perform a complex synchronization to make sure the change is reflected in every range. Right now, that synchronization involves propagating information in 20 different directions, Yu said. This is necessary because the code doesn't track which fields have been updated, so synchronizing the bounds after processing a chunk of code involve sharing information from each of the five tracked constraints (four ranges and a `tnum`), to each of the other four. 

Instead of tracking ranges in the current way, Yu proposes tracking ranges using a variant of the approach he [discussed](<https://lwn.net/ml/all/ZTZxoDJJbX9mrQ9w@u94a/>) in October 2023\. Essentially, the maximum would be allowed to be lower than the minimum. The range represented this way always starts at the minimum and ends at the maximum, but it might wrap around part way through. This means that the range (minimum: 0xFFFFFFFC, maximum: 4) represents the signed range (-4, 4) while simultaneously representing the _unsigned_ ranges (0xFFFFFFFC, UINT32_MAX) and (0, 4). The existing code doesn't handle disjoint ranges like that, so Yu plans to add some conversion functions that convert from the new representation for use by the old code. 

Storing ranges this way has a few benefits. The biggest one is that there is no need to synchronize knowledge of signed and unsigned bounds — they are automatically synchronized, simply by virtue of the representation. That also cuts down on the amount of information the verifier needs to propagate between the known-bits representation and the range representations, bringing the code down to only six directions of information flow (from each of the three bounds to the two others). Yu hopes that this will make the verifier code that handles value tracking much easier to work with, and also to formally verify. 

Yu plans to get the changes into upstream in a few steps; initially, there will be conversion functions and the main verifier code will remain largely unchanged. Then, he plans to change the most critical parts of the value-tracking code to use the new representation natively, followed by adapting the kernel selftests. Finally, the last uses can be removed along with the conversion functions. 

Yu's second proposal for simplifying the value tracker is to introduce a more abstract interface for working with `tnum` and range values. The proposals can be implemented independently, but they would certainly complement one another. Right now, working on the verifier code requires knowledge of the internal details of `tnum`s and ranges; but the most common operations to perform on these values are just intersections and inclusion checks, Yu said. If those were pulled out into their own functions, a lot of the actual value-tracking code could be substantially simplified. 

Those aren't the only possible ways to ease maintenance of the verifier, however. The session ended with a discussion of how to improve the documentation, what aspects of the verifier could potentially be standardized, and how these proposals would impact formal verification. The verifier has certainly earned its reputation as a tricky piece of code to maintain, but it seems like the kernel's BPF developers have a plan to start changing that.
