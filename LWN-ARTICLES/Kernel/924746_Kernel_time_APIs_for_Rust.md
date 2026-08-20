---
title: Kernel time APIs for Rust
url: https://lwn.net/Articles/924746/
date: "March 2, 2023"
category: "Development tools-Rust; Timekeeping"
author: "By Jonathan Corbet March 2, 2023"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 2, 2023

While the 6.3 kernel has gained more support for the Rust language, it still remains true that there is little that can be done in Rust beyond the creation of a "hello world" module. That functionality was already available in C, of course, with a level of safety similar to what Rust can provide. Interest is growing, though, in merging actually useful modules written in Rust; that will require some more capable infrastructure than is currently present. A recent discussion on the handling of time values in Rust demonstrates the challenges — and opportunities — inherent in this effort. 

Asahi Lina, who is [implementing a graphics driver](<https://lwn.net/ml/rust-for-linux/70657af9-90bb-ee9e-4877-df4b14c134a5@asahilina.net/>) for Apple hardware in Rust, has posted a number of pieces of Rust infrastructure, including [a module for timekeeping functions](<https://lwn.net/ml/linux-kernel/20230221-gpu-up-time-v1-1-bf8fe74b7f55@asahilina.net/>). The timekeeping module itself is just a beginning, weighing in at all of 25 lines; it looks like this: 

```
// SPDX-License-Identifier: GPL-2.0
        
        //! Timekeeping functions.
        //!
        //! C header: [`include/linux/ktime.h`](../../../../include/linux/ktime.h)
        //! C header: [`include/linux/timekeeping.h`](../../../../include/linux/timekeeping.h)
        
        use crate::bindings;
        use core::time::Duration;
        
        /// Returns the kernel time elapsed since boot, excluding time spent
        /// sleeping, as a [`Duration`].
        pub fn ktime_get() -> Duration {
            // SAFETY: Function has no side effects and no inputs.
            Duration::from_nanos(unsafe { bindings::ktime_get() }.try_into().unwrap())
        }
        
        /// Returns the kernel time elapsed since boot, including time spent
        /// sleeping, as a [`Duration`].
        pub fn ktime_get_boottime() -> Duration {
            Duration::from_nanos(
                // SAFETY: Function has no side effects and no variable inputs.
                unsafe { bindings::ktime_get_with_offset(bindings::tk_offsets_TK_OFFS_BOOT) }
                    .try_into()
                    .unwrap(),
            )
        }
```

This module expresses two kernel functions — [`ktime_get()`](<https://elixir.bootlin.com/linux/v6.2.1/source/kernel/time/timekeeping.c#L836>) and [`ktime_get_boottime()`](<https://elixir.bootlin.com/linux/v6.2.1/source/include/linux/timekeeping.h#L87>) — as Rust equivalents that return values as the Rust [`Duration`](<https://doc.rust-lang.org/stable/std/time/struct.Duration.html>) type. In C, these functions both return a `ktime_t`, which is a signed, 64-bit value reflecting a time in nanoseconds. The origin of that time — what real-world date and time is represented by a `ktime_t` of zero — varies depending on which clock is being queried. In the case of `ktime_get_boottime()`, for example, the returned value represents the time that has passed since the system booted. 

Kernel times are thus, at their core, a delta value; they are a count of nanoseconds since some beginning-of-the-universe event. The proposed Rust implementation followed that lead in its use of the `Duration` type. But Thomas Gleixner, who is responsible for much of the kernel's timekeeping code, [questioned](<https://lwn.net/ml/linux-kernel/87v8jvnqq4.ffs@tglx/>) this approach. Since both of the functions are meant, each in its own way, to represent an absolute point in time, he suggested that the Rust functions should return an absolute-time type; he suggested either [`Instant`](<https://doc.rust-lang.org/std/time/struct.Instant.html>) or [`SystemTime`](<https://doc.rust-lang.org/std/time/struct.SystemTime.html>). Both represent absolute time values; `Instant` is monotonic (it will never go backward) while `SystemTime` is not. 

That approach will not work well in the kernel, though, for a couple of reasons. The first, as [pointed out](<https://lwn.net/ml/linux-kernel/Y%2FTP6as7qqwfcI42@boqun-archlinux/>) by Boqun Feng, is that those two types are defined in [the Rust standard library ("`std`")](<https://doc.rust-lang.org/std/>), which, just like the C standard library, is not available in the kernel. So, at best, those two types would have to be reimplemented for kernel use. But the other problem is that the kernel supports a multitude of clocks; the list can be found in the [`clock_gettime()` man page](<https://man7.org/linux/man-pages/man3/clock_gettime.3.html>). Each clock has a reason for existing, and each behaves a little differently. A type like `Instant` is defined to use exactly one clock, but kernel code will need access to several of them. 

Gleixner [was not really concerned](<https://lwn.net/ml/linux-kernel/87h6vfnh0f.ffs@tglx/>) about the exact types used, but he did call for different types to be used for absolute times (timestamps) and delta times (or intervals). The kernel does not currently have such a distinction in its types for times, but libraries for many languages do make that distinction. Given that Rust is being brought into the kernel in the hope of making it easy to write safer code, it makes sense to use Rust's type system to prevent developers from, for example, trying to add two absolute-time values together. 

Indeed, as Lina [pointed out](<https://lwn.net/ml/linux-kernel/7b93bf74-abdc-f8c1-9a12-7c7f080f9e19@asahilina.net/>), type safety in the Rust interface should even go one step further. Subtracting one absolute time from another will yield a delta time — but that delta time will only make sense if the two absolute times came from the same clock. So the type system should prevent the incorrect mixing of times from different clocks. 

What about delta times? Gleixner initially [suggested](<https://lwn.net/ml/linux-kernel/87edqioo1e.ffs@tglx/>) that time deltas could be independent of any clock; a time delta obtained by subtracting one `CLOCK_BOOTTIME` value from another would be the same type as a delta calculated as a difference of `CLOCK_TAI` values. Heghedus Razvan [agreed](<https://lwn.net/ml/linux-kernel/p7e4M5-kxxScH56bBMtlzht2_m33oSfMPHtExSVbDn0WIPYBEbiJjr2NvNhqCJotFhmYqH5h27EWPDhfSmmST0D-dSU1Alq4_BzDQUR48FA=@protonmail.com/>) with this idea and posted a sample implementation; Gary Guo then [polished that idea](<https://lwn.net/ml/linux-kernel/20230222195537.3462c1c3.gary@garyguo.net/>) into [a "more Rusty" implementation](<https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=83702f491050da1c67ab9aa129103f7d>). Miguel Ojeda, though, [suggested](<https://lwn.net/ml/linux-kernel/CANiq72n-++roTv3yDNOA=Mi_sscBZX3xPdZ6RZ0Oxx+T82pf5A@mail.gmail.com/>) that delta times, too, could be tied to a specific clock. Gleixner [was not entirely convinced](<https://lwn.net/ml/linux-kernel/87o7pmmtre.ffs@tglx/>) that this distinction was needed, but agreed that there might be value in it, especially when dealing with timers. Kernel timers, too, can be based on specific clocks, so it might make sense to ensure that any time deltas used with those timers are generated with the same clock, he said. 

Feng [suggested](<https://lwn.net/ml/linux-kernel/Y%2FWDz013yBIfQPDq@boqun-archlinux/>) proceeding with an implementation using `Duration` for all time delta values and something resembling `Instant`, but with clock-specific typing, for absolute times. Lina [agreed](<https://lwn.net/ml/linux-kernel/03e9af69-3fea-032d-6dc3-ac0ea8e57471@asahilina.net/>), and seems ready to proceed in this direction, starting with the example posted by Razvan. A new patch will, presumably, be forthcoming. 

It seems likely that we will see this sort of conversation happening repeatedly as more Rust infrastructure heads toward the mainline. It is certainly possible to reproduce something like the existing kernel APIs in Rust, and doing so would make the resulting Rust code look more familiar to current kernel developers. But that approach also risks losing much of the hoped-for benefit that is driving the Rust experiment in the first place. Doing this experiment properly will require showing how Rust can lead to the creation of better, safer APIs than what the kernel has now. So a lot of APIs are going to have to be redesigned from the beginning; they can benefit from years of experience in the kernel community, but will have to leave many of that community's conventions behind. It seems like a project that will keep people busy for some time.
