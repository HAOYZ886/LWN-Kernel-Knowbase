---
title: Exceptions in BPF
url: https://lwn.net/Articles/938435/
date: "July 21, 2023"
category: "BPF; Releases-6.7"
author: "By Jonathan Corbet July 21, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 21, 2023

The BPF virtual machine in the kernel has been steadily gaining new features for years, many of which add capabilities that C programmers do not ordinarily have. So, from one point of view, it was only a matter of time before BPF gained support for exceptions. As it turns out, though, this "exceptions" feature is aimed at a specific use case, and its use in most programs will be truly exceptional. 

Kumar Kartikeya Dwivedi posted [the BPF exceptions patch set](<https://lwn.net/ml/bpf/20230713023232.1411523-1-memxor@gmail.com/>) on July 13\. The API presented to BPF programs is simple, taking the form of two [kfuncs](<https://lwn.net/Articles/856005/>). To raise an exception, a BPF program can call: 

```
void bpf_throw(u64 cookie);
```

A call to `bpf_throw()` will cause the program's entire call stack to be unwound, and the program will return to its caller with (by default) a return status of zero; the `cookie` value is ignored. There is no way for a program to catch an exception called further down the call stack. It is, however, possible to define a function to be called after the call stack has been unwound, but before control is returned to the caller: 

```
void bpf_set_exception_callback(int (*callback)(u64));
```

The given `callback()` will be called once unwinding is complete, and will be passed the `cookie` value given to `bpf_throw()`; its return value will then be returned to the original caller of the BPF program. There can be only one `bpf_set_exception_callback()` call in a program; once the callback is set, it cannot be changed. 

One might thus be forgiven for thinking that this exception mechanism does not look like it does in other languages supporting the feature, and that `bpf_throw()` might better be spelled `exit()`. It clearly is not meant to allow BPF programs to catch and respond to unusual situations. The use case for exceptions, as it turns out, is different and unique to BPF. 

BPF programs must, famously, convince the kernel's verifier that they are safe to run before they can be successfully loaded. Doing so requires handling every possible case — even cases that the programmer knows can never happen, but which the verifier is less certain about. So, for example, a BPF function far down the call stack might have to check that an integer value is within a given range, even though the developer knows that it must be, because the verifier does not know that. The check must do something reasonable in response to an out-of-bounds value and, perhaps, return a failure status all the way back up the call chain, all for a case that can never happen. 

And, as we all know, developers are never wrong about cases that can never happen. 

As Dwivedi [described](<https://lwn.net/ml/bpf/CAP01T77m9ArXiKLgKW-uttFikirC97VXP=hmXFkS8JrSDXRYJg@mail.gmail.com/>), exceptions are intended to address this problem: 

> The primary requirement was for implementing assertions within a program, which when untrue still ensure that the program terminates safely. Typically this would require the user to handle the other case, freeing any resources, and returning from a possibly deep callchain back to the kernel. Testing a condition can be used to update the verifier's knowledge about a particular register. 

So, in other words, the real reason for exceptions is to provide a mechanism by which the verifier can be informed of invariants that the developer knows about while having an emergency exit mechanism for those times when the developer is wrong. There is [a set of assertion macros](<https://lwn.net/ml/bpf/20230713023232.1411523-10-memxor@gmail.com/>) provided to make this feature easily available in BPF programs. So, for example, a developer will be able to write: 

```
bpf_assert_lt(foo, 256);
```

This assertion will perform the indicated test and, should it fail, make a call to `bpf_throw()`. Meanwhile, the verifier will be able to use the knowledge that `foo` is, indeed, less than 256 as it evaluates the subsequent code. 

There is one notable problem still, as described in the changelog to [this patch](<https://lwn.net/ml/bpf/20230713023232.1411523-7-memxor@gmail.com/>) in the series: 

> For now, bpf_throw invocation fails when lingering resources or locks exist in that path of the program. In a future followup, bpf_throw will be extended to perform frame-by-frame unwinding to release lingering resources for each stack frame, removing this limitation. 

Given that the verifier is now counting on `bpf_throw()` to prevent execution from proceeding past a failed assertion, this seems like a significant limitation indeed. It could probably be used by a sufficiently malicious developer to convince the verifier to accept a program that does something unpleasant. That suggests that implementing the frame-by-frame unwinding will be a prerequisite to getting this work merged. 

Both BPF and Rust are intended to make kernel programming safer, but they take a different approach to the problem. A Rust program will, by default, panic if any of a large number of things goes wrong. BPF programs, instead, are intended to be verified as simply lacking that sort of wrong behavior before they are ever allowed to execute. BPF exceptions can be seen as an admission that the "prove correctness before loading" approach has its limits, and that sometimes it is necessary to just throw up your hands and bail out.
