---
title: Smart pointers for the kernel
url: https://lwn.net/Articles/992055/
date: "October 4, 2024"
category: "Development tools-Rust"
author: "By Daroc Alden October 4, 2024 Kangrejos 2024"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Daroc Alden**  
October 4, 2024

* * *

[Kangrejos 2024](<https://lwn.net/Articles/990496/>)

Rust has a plethora of smart-pointer types, including reference-counted pointers, which have special support in the compiler to make them easier to use. The Rust-for-Linux project would like to reap those same benefits for its smart pointers, which need to be written by hand to conform to the [ Linux kernel memory model](<https://lwn.net/Articles/718628/>). Xiangfei Ding presented at Kangrejos about the work to enable custom smart pointers to function the same as built-in smart pointers. 

Ding showed the specific "superpowers" that built-in smart pointers have in his [ slides](<https://kangrejos.com/2024/SmartPointer%20and%20PinCoerceUnsized.pdf>): unsizing and dynamic dispatch. Unsizing allows the programmer to remove the length of an array behind a pointer from its type, turning a `Ptr<[T; N]>` (bounds-checked at compile time) into a `Ptr<[T]>` (bounds-checked at run time). This needs special support because slices (values of type `[T]`) do not have a known size at compile time; therefore the compiler needs to store the size somewhere at run time. The compiler could store the size in the pointed-to allocation, but that would require reallocating the array's memory, which would be expensive. Instead, the compiler stores the size alongside the pointer itself, as a fat pointer. On nightly Rust compilers, users can enable an experimental feature and then have their pointer type implement [`CoerceUnsized`](<https://doc.rust-lang.org/std/ops/trait.CoerceUnsized.html>) to indicate that it supports that. 

[ ![\[Xiangfei Ding\]](https://static.lwn.net/images/2024/xiangfei-ding-small.png) ](<https://lwn.net/Articles/992336>)

The second superpower is called [`DispatchFromDyn`](<https://doc.rust-lang.org/std/ops/trait.DispatchFromDyn.html>) and allows converting a `Ptr<T>` into a `Ptr<dyn Trait>` when `T` implements `Trait`. This has to do with the way that Rust implements dynamic dispatch — a value of type `Ptr<dyn Trait>` uses a dispatch table to find the implementation of the method being invoked at run time. That method expects to receive a `self` pointer. So converting a smart pointer to use dynamic dispatch only works when the smart pointer can be used as a `self` pointer. 

These features are both experimental, because the Rust project is still working on their design. Ding explained that there is an [ RFC](<https://rust-lang.github.io/rfcs/3621-derive-smart-pointer.html>) aimed at stabilizing just enough for the Linux kernel to use, without impeding the development of the features. The RFC would add a new macro that makes it trivial for a smart pointer satisfying certain requirements to implement the necessary traits, no matter what the final forms of the traits end up looking like. That would let the kernel start using its custom smart pointers on stable Rust sooner rather than later. 

There is one catch — implementing these features for a smart-pointer type with a malicious or broken [ `Deref`](<https://doc.rust-lang.org/std/ops/trait.Deref.html>) (the trait that lets a programmer dereference a value) implementation could break the guarantees Rust relies on to determine when objects can be moved in memory. This is of particular importance to [`Pin`](<https://doc.rust-lang.org/std/pin/struct.Pin.html>), which is a wrapper type used to mark an allocation that cannot be moved. It's not hard to write smart-pointer types that don't cause problems, but in keeping with Rust's commitment to ensuring safe code cannot cause memory-safety problems, the RFC also requires programmers to use unsafe (specifically, implementing an `unsafe` [ marker trait](<https://users.rust-lang.org/t/understanding-the-marker-traits/75625/3>)) as a promise that they've read the relevant documentation and are not going to break `Pin`. With that addition, the code for a smart-pointer type would look like this: 

```
// Use Ding's macro ...
        #[derive(SmartPointer)]
        // On a struct that is just a wrapper around a pointer
        #[repr(transparent)]
        struct MySmartPointer<T: ?Sized>(Box<T>);
    
        // Implement Deref, with whatever custom logic is needed
        impl<T: ?Sized> Deref for MySmartPointer<T> {
            type Target = T;
            fn deref(&self) -> &T {
                ...
            }
        }
    
        // And then promise the compiler that the Deref implementation is okay to
        // use in conjunction with Pin:
        unsafe impl<T: ?Sized> PinCoerceUnsized for MySmartPointer<T> {}
```

Andreas Hindborg asked for some clarification about why the marker trait is needed. `Deref` is supposed to be simple, Ding explained. Usually, someone writing a smart-pointer type would have a normal pointer stored in their type; when implementing `Deref`, they can just use the normal pointer. But it's technically possible to implement something more complicated than that. In this case, you could have a `Deref` implementation that actually moves data out of the object pointed to and stores something else there. This would not normally be a problem, except when the smart pointer is contained in a `Pin`, which is supposed to prevent the value from being moved. If the `Deref` implementation moves the value anyway, then that would be undefined behavior. The unsafe marker trait is a promise to the compiler that the programmer has not done that. 

The new macro is available on nightly Rust, although Ding says that it needs a bit more testing in order to stabilize, as well as some additional documentation which he is working on. Miguel Ojeda asked how soon the macro might be stabilized; Ding answered that it should be quite soon. He will make a stabilization report shortly, and then it is just a matter of checking off the requirements.
