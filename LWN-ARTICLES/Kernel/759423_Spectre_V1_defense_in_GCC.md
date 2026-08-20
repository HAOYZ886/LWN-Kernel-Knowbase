---
title: Spectre V1 defense in GCC
url: https://lwn.net/Articles/759423/
date: "July 10, 2018"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet July 10, 2018"
---

By **Jonathan Corbet**  
July 10, 2018

In many ways, Spectre variant 1 (the bounds-check bypass vulnerability) is the ugliest of the Meltdown/Spectre set, despite being relatively difficult to exploit. Any given code base could be filled with V1 problems, but they are difficult to find and defend against. Static analysis can help, but the available tools are few, mostly proprietary, and prone to false positives. There is also a lack of efficient, architecture-independent ways of addressing Spectre V1 in user-space code. As a result, only a limited effort (at most) to find and fix Spectre V1 vulnerabilities has been made in most projects. An effort to add some defenses to GCC may help to make this situation better, but it comes at a cost of its own. 

Spectre V1, remember, comes about as the result of an incorrect branch prediction by the processor. Given code like: 

```
if (index < structure->array_size)
            do_something_with(structure->array[index]);
```

The processor would likely predict that `index` would indeed be less than the given size since, in normal execution, it almost always is. It will then go on to speculatively execute the code that uses `array[index]` with an `index` value that may, instead, be far out of bounds. If this speculative access leaves traces elsewhere in the system (by pulling data into the cache, for example), it can be exploited to leak data that, in a correct execution of the code, would be protected. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

In the kernel, the `array_index_nospec()` macro has been [introduced](<https://lwn.net/Articles/746551/>) as a way to prevent incorrect speculative loads of this type. These macro calls must be introduced manually, though, in places where somebody has determined that a Spectre V1 vulnerability may exist. That has been happening, but slowly; there are about 60 invocations in the 4.18-rc4 kernel. Less work has been done in user space, though, for a number of reasons, including the lack of a primitive like `array_index_nospec()`. 

GCC may soon address that final problem, thanks to [this patch set](<https://lwn.net/Articles/759438/>) from Richard Earnshaw, based on a technique first [published](<https://docs.google.com/document/d/1wwcfv3UV9ZnZVcGiGuoITT_61e_Ko3TmoCS3uXLcJR0/edit#heading=h.phdehs44eom6>) by Chandler Carruth. These patches add a new intrinsic that behaves much like `array_index_nospec()`: 

```
__builtin_speculation_safe_value(value, fallback)
```

In the absence of speculation, this function will simply return `value`. When speculative execution is happening, instead, it _might_ still return `value`, but it could also return the `fallback` value, which defaults to zero. It can thus be used to ensure that speculative execution cannot happen with out-of-range index values. A simple implementation would just use a barrier unconditionally to prevent speculation outright, but barriers can be expensive. It may be more efficient to just clamp the range of the index value while allowing speculation in general to continue. 

#### Detecting incorrect speculation

A look at how this new intrinsic works yields some insight into why it is specified the way it is. The core of that implementation is a trick to detect when incorrect speculative execution is occurring and to prevent out-of-bounds accesses from happening in such situations. Doing so requires instrumenting the code as it is built by the compiler. In this scheme, the above `if` statement would be modified to look something like this: 

```
void *all_ones = ~0;
        void *all_zeroes = 0;
        void *correct = all_ones;
    
        if (index < structure->array_size) {
            correct = (index >= structure->array_size) ? all_zeroes : correct;
    	index &= correct;
    	do_something_with(structure->array[index]);
        }
```

The key is the assignment of `correct` inside the body of the `if`: 

```
correct = (index >= structure->array_size) ? all_zeroes : correct;
```

That assignment tests whether the _inverse_ of the branch condition is true; if that is the case, the body is being speculatively executed when it should not be and evasive action is required. Since `correct` will have been set to zero if (and only if) incorrect speculation is taking place, said evasive action can take the form of using `correct` as a mask against `index`: 

```
index &= correct;
```

In normal execution, this operation will change nothing; when incorrect speculative execution has been detected, instead, `index` will be reset to zero. At that point, it can no longer be used to speculatively access out-of-bounds memory. 

The question that may come to mind here is: if the condition is mispredicted in the `if` statement, won't the same thing happen with the ternary expression used to set the value of `correct`? As it happens, almost all architectures have some sort of compare-and-assign operation that (1) is a single instruction without a branch, so the branch predictor does not enter the picture, and (2) is defined by the architecture to not be subject to speculation in its own right. So the assignment of `correct` will be done with non-predicted values; it will be an accurate indicator of whether incorrect speculative execution is taking place. 

Note that the `correct` flag is initialized once, but updated after every branch as shown above. It will, thus, carry the prediction state through multiple branches if need be. With enough cleverness, it can even be used to communicate this state across function calls. Since speculation can sometimes run hundreds of instructions ahead of anything known to be correct, this ability to track and communicate the state of execution is important. 

#### Adding support to GCC

As noted above, implementing `__builtin_speculation_safe_value()` can be as simple as injecting a barrier into the generated code. But if the compiler could also add the ability to detect incorrect speculation, other possibilities would open up. To that end, the GCC patch set under consideration adds a new `-mtrack-speculation` option for compilation that turns on this mechanism. [This patch](<https://lwn.net/ml/gcc-patches/1531154299-28349-7-git-send-email-Richard.Earnshaw@arm.com/>), in particular, adds speculation tracking for the arm64 architecture. As described in that patch, a simple equality test might (after the comparison to set the condition code) look like: 

```
B.EQ	<dst>
            ...
        <dst>:
```

With `-mtrack-speculation`, that code would be made to look more like this: 

```
B.EQ	<dst>
    	CSEL	tracker, tracker, XZr, ne
            ...
        <dst>:
            CSEL	tracker, tracker, XZr, eq
```

Here, `tracker` is the name of the register that has been dedicated to holding the `correct` flag. The [`CSEL` instruction](<http://infocenter.arm.com/help/topic/com.arm.doc.dui0802a/CSEL.html>) will set `tracker` to either itself or `XZr` (the register holding all zeroes) depending on the real value of the condition, without speculation. It is, in other words, implementing the ternary operator we saw in the example above. 

This operation will cause the `tracker` register to be zero when incorrect speculation is happening. That allows it to be used to implement `__builtin_speculation_safe_value()`; with the default `fallback` value of zero, a logical AND between the `tracker` register and the value in question will suffice. In the case of the arm64 architecture, though, it is possible to do a little better. When speculation tracking is turned on, the compiler will simply insert a `CSDB` speculation barrier when incorrect speculation is detected. 

It's worth noting in passing that things become more complicated when function calls are involved. Speculative execution can involve function calls, so it is important to track incorrect speculation across those calls. If a register could be dedicated program-wide to the `tracker` value, life would be easy, but that would require a flag-day change to the arm64 ABI. Instead, the stack pointer is used in a tricky way to encode the correctness state on function call and return; see the above-linked patch for details. 

Overall, this approach may seem like the best of all worlds; barriers can be expensive, so a mechanism that only executes them when they are known to be necessary would be ideal. The downside, of course, is that the speculation tracking itself is not cheap. It requires setting aside two registers to track the state and the instrumentation of every branch. No benchmark results have been posted with the code, but this level of overhead must have an impact. The cost is high enough to rule out otherwise interesting ideas like automatically protecting all bounds checks. 

In any case, this sort of speculation tracking may come across as a strange mechanism; code running on the processor can detect that the processor has speculated incorrectly, but the processor itself still takes some time to figure that out. But that is the world we have found ourselves living in. The best that can be done is to find ways of protecting our code while minimizing the cost.
