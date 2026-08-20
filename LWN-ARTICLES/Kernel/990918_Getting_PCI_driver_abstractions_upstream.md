---
title: Getting PCI driver abstractions upstream
url: https://lwn.net/Articles/990918/
date: "September 26, 2024"
category: "Development tools-Rust"
author: "By Daroc Alden September 26, 2024 Kangrejos 2024"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Daroc Alden**  
September 26, 2024

* * *

[Kangrejos 2024](<https://lwn.net/Articles/990496/>)

Danilo Krummrich gave a talk at Kangrejos 2024 focusing on the question of how the Rust-for-Linux project could improve at getting device and driver abstractions upstream. As a case study, he used some of his recent work that attempts to make it possible to write a PCI driver entirely in Rust. There wasn't time to go into as much detail as he would have liked, but he did demonstrate that it is possible to interface with the kernel's module loader in a way that is much harder to screw up than the current standard approach in C. 

To give context to the discussion, he started by explaining that his goal was to make development of Nova (the [ new NVIDIA driver he has been working on](<https://lwn.net/Articles/990736>)) go smoothly. He opined that Nova would probably be the first "more complex" thing to go upstream. Luckily, other in-progress efforts to write kernel components in Rust need some of the same abstractions as Nova, including [ rvkms](<https://lwn.net/ml/all/20240322221305.1403600-1-lyude@redhat.com/>), [ rnvme](<https://rust-for-linux.com/nvme-driver>), [ Apple AGX](<https://lwn.net/Articles/988438/>), and [ rcpufreq-dt](<https://lwn.net/ml/all/cover.1722334569.git.viresh.kumar@linaro.org/>). Ultimately, he would like to provide driver infrastructure that integrates with the kernel, but that takes advantage of Rust's capabilities where possible. 

Initially, there was confusion on the mailing list, Krummrich said, about what the Rust abstractions he wanted to discuss were meant to represent. [ The patch set](<https://lwn.net/ml/all/20240618234025.15036-1-dakr%40redhat.com/>) that spawned the discussion is not really about abstracting _drivers_ he explained, but about permitting drivers to access bus types safely — abstractions needed _for_ drivers. A fact which had proved difficult to communicate via email. At the end of the discussion, Greg Kroah-Hartman had called for writing the driver-registration code itself separately, in C. 

At Kangrejos, Kroah-Hartman took responsibility for half of the misunderstanding, but said that he did still think it made sense to implement driver registration itself in C. He said that it would be a small amount of "safe C" — a comment that prompted good-natured chuckles from the attending Rust programmers. Krummrich was sympathetic, but wanted to go through an example of what he was trying to achieve with the driver abstractions and why. 

Krummrich showed the code for a simple PCI driver written in C, and then went through a series of modifications to incrementally rewrite it in Rust. The simplest change was just to replace the probe and remove functions with versions written in Rust: 

```
#[no_mangle]
        unsafe extern "C" fn rust_pci_driver_probe(
            _pdev: *mut bindings::pci_dev,
            _ent: bindings::pci_device_id,
        ) -> core::ffi::c_int {
            pr_info!("Probe Rust PCI driver sample.\n");
    
            0
        }
    
        #[no_mangle]
        unsafe extern "C" fn rust_pci_driver_remove(_pdev: *mut bindings::pci_dev) {
            pr_info!("Remove Rust PCI driver sample.\n");
        }
```

The code was fairly simple, but after showing it, Krummrich explained that there was actually already a bug in this example that he hadn't noticed until later: the signature of the Rust functions was incorrect. It didn't cause problems, because the incorrect parameter was unused, but it still served to illustrate why he wanted to make interfacing with the kernel's driver system less error-prone. 

He also mentioned an issue that didn't show up in this simple example — binding object lifetimes to driver and module lifetimes, instead of _device_ lifetimes, such as is sometimes necessary for complex drivers. Kroah-Hartman thought that it was good for that to be hard, because "we don't want you to do that". He explained that the kernel tried hard to ensure that data is bound to data, not to code, and that drivers should use per-device storage. Krummrich said that the same problem applied to trying to set up per-device storage, in this case. Ultimately, they both agreed that what they wanted was simple lifetime handling that bound data (such as cache data) to other data (such as a specific device), and that the simplest way to integrate Rust and C code did not make that easy. 

Krummrich did include the caveat that Nova had some buffers that needed to live for the lifetime of the module itself, because of how NVIDIA's GPU system processor handles debugging information. Kroah-Hartman replied that GPUs were "the most crazy complex hardware out there". So he was fine with GPU drivers needing to do something out of the ordinary — but normal drivers shouldn't need to. 

This prompted some discussion of which subsystems might need to do things differently. Kroah-Hartman agreed that some of the core infrastructure would need more complicated lifetime management as well. Most drivers are simple I2C or PCI drivers, however, he said. So it's only a handful of complicated drivers that need more complex lifetime management. 

Krummrich then showed how the bindings could be made safer by moving some of the implementation into Rust; he demonstrated this in stages. The first stage was to use a macro to declare the module in Rust instead of C. That still left unsafe functions, the possibility that the driver could forget to unregister correctly, and some awkwardness with constructing the PCI device ID table. 

Next, he showed how the uses of unsafe could be centralized to a single helper library, such that each driver only needs safe functions, with signatures type-checked by the compiler. The new version involves creating a type that implements the `pci::Driver` trait, which then takes care of the details of registering and unregistering the driver, and storing device-specific information in a type-safe way. 

There were some more questions about the specific details. Kroah-Hartman asked why Krummrich had reimplemented a structure in Rust instead of using the version defined in C; Krummrich said that he didn't want to expose the structure directly to drivers. Benno Lossin suggested using a type alias, so that it's still not defined in two places. 

The most involved part of the code concerned the machinery to make the PCI device ID table automatically add an appropriate sentinel value to the end at compile time, so that the whole device ID table can be stored in the binary's data section — a much more elegant solution than creating a copy of the table at run time. That ended up requiring the use of some additional intermediate traits, and made the whole code more than twice as large. 

Kroah-Hartman said that the code was great, but that the complexity needed to add the sentinel "sounds crazy". He asked whether there wasn't a simpler way to do it, but nobody present volunteered a way. 

Krummrich's example did show that it is definitely possible to build a driver-binding abstraction that requires no unsafe code in drivers. Such a driver cannot accidentally forget to unregister, automatically has type-safe storage associated with each device that it is registered for, does not need to touch raw pointers (instead receiving bounds-checked smart pointers for the appropriate PCI buses), and should generally be quite difficult to misuse. 

The session ran out of time before Krummrich could finish his presentation. The overall conclusion from Kroah-Hartman was that he wants to see the full example in a patch set, but that it was a promising start. Krummrich wants this driver abstraction to be available for Nova and the other Rust drivers that use the same features, so it seems likely that he'll provide that example soon.
