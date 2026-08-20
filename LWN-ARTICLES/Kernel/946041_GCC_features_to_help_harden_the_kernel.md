---
title: GCC features to help harden the kernel
url: https://lwn.net/Articles/946041/
date: "October 5, 2023"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet October 5, 2023 Cauldron"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 5, 2023

* * *

[Cauldron](<https://lwn.net/Archives/ConferenceByYear/#2023-GNU_Tools_Cauldron>)

Hardening the Linux kernel is an endless task, with work required on multiple fronts. Sometimes, that work is not done in the kernel itself; other tools, including compilers, can have a significant role to play. At the [2023 GNU Tools Cauldron](<https://gcc.gnu.org/wiki/cauldron2023>), Qing Zhao covered some of the work that has been done in the GCC compiler to help with the hardening of the kernel — along with work that still needs to be done. 

The [Kernel self-protection project](<https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project>) is the home for much of the kernel-hardening work, she began. Hardening can be done in a number of ways, starting with the fixing of known security bugs, which may be found by static checkers, fuzzers, or code inspection. Fixing bugs is a never-ending task, though; it is far better, when possible, to eliminate whole classes of bugs entirely. Thus, much of the work in the kernel has focused on getting rid of problems like stack and heap overflows, integer overflows, format-string injection, pointer leaks, use of uninitialized variables, use-after-free bugs, and more. Effort is also going into blocking methods of exploitation, including the ability to overwrite kernel text or function pointers. 

[![\[Qing Zhao\]](https://static.lwn.net/images/conf/2023/gtc/QingZhao-sm.png)](<https://lwn.net/Articles/946045/>) The GCC 11 release (April 2021), she said, included the ability to zero the registers used by a function on return from that function; that can help prevent the leakage of information. It is now on by default. GCC 12 (May 2022), instead, added the automatic initialization of stack variables; that, too, has been turned on by default in kernel builds. GCC 13 (April 2023) added more strict treatment of flexible-array members. 

Zhao briefly mentioned some of the features that the kernel community would like to see in future compiler releases. These include better support for flexible-array checking, a reduction in false-positive warnings with the `‑warray‑bounds` option, better integer-overflow checking, support for control-flow integrity checking, and more. 

Returning to flexible arrays, Zhao pointed out that out-of-bounds array accesses are a major source of vulnerabilities in the kernel. These can be prevented by bounds checking — if the size of the array in question is known. For fixed arrays, the size is known at compile time, so the checking of array accesses can be done, either at compile time (if possible) or at run time. For dynamically sized arrays, though, the problem is harder. In C, these arrays take two forms: variable-length arrays and flexible-array members in structures; only the latter are used in the kernel at this point. 

A flexible-array member is an array embedded within a structure as the final element. It is often declared as having a dimension of either zero or one (though the latter tends to be a frequent source of bugs), or just as `array[]`. When space for an instance of the structure is allocated, it must be sized large enough to hold the actual array, which will vary in length from one instance to the next. 

In GCC 12, all arrays that are defined as the final member of a structure are considered to be flexible, regardless of the declared size of the array. So even the array here: 

```
struct foo {
            int int_field;
    	int array[10];
        };
```

would be deemed by the compiler to be flexible in size even though that was (probably) not the developer's intent; as a result, no bounds checking is performed on accesses to those arrays. In GCC 13, the `‑fstrict‑flex‑arrays` option gives control over which arrays are considered to be flexible arrays; [this article](<https://lwn.net/Articles/908817/>) gives an overview of how it works. The result is that bounds checking can be more easily applied to arrays that were never meant to vary in size. 

There are still some problems, though; Zhao mentioned the case of a structure containing a flexible-array member that is, in turn, nested into another structure type: 

```
struct s1 {
            int flex_array[0];
        };
    
        struct s2 {
    	type_t some_field;
    	struct s1 flex_struct;
        }
```

Even if the flexible structure is the final member of the containing structure (`s2` above), versions of GCC less than 14 will incorrectly treat the array as fixed. Zhao has contributed a fix for that particular problem. A separate problem arises when the flexible structure is _not_ the final field of the containing structure. In this case, it's not clear what the compiler should do, but GCC has accepted such structures. The new `‑Wflex‑array‑member‑not‑at‑end` option will warn about such code. 

Flexible-array members in unions are yet another problem; GCC will accept such members when declared as `array[0]`, but the (legal) `array[]` form is not accepted. That makes it impossible to create unions that will compile under the strictest `‑fstrict‑flex‑array` mode. Unions containing _only_ flexible array members raise a different issue: they could end up being a zero-length object, which is not something the C standard allows. Adding a fixed-length member resolves that issue for now; there may be an attempt to allow fully flexible unions as a future GCC extension. 

Use of flexible arrays currently defeats bounds checking, but the actual length of any given array is (or at least should be) known to the code as it is running. If that size can be communicated to the compiler, bounds checking can be added. There are two potential ways of declaring that information; one would be to add a new syntax to embed it within the type itself: 

```
struct foo {
            size_t len;
    	char array[.len*4];
        };
```

This syntax allows the use of expressions (in this case, "four times the value of the `len` field"). It is the cleaner option, she said, but it has the potential to break ABIs for existing code by changing the dimension of the array. That makes it harder to adopt, as does the syntax change, which is sure to require a lot of discussion before it would find acceptance. 

An alternative is to add an attribute to the flexible-array member instead. That preserves the existing ABI, is easier to adopt, and can also be extended to other types (pointers, for example). On the other hand, it is harder to extend to more complex expressions. The [`counted_by()` attribute](<https://lwn.net/Articles/936728/>) has been added for GCC 14 without expression support; it can only refer to another field in the same structure for now. 

```
struct foo {
            size_t len;
    	char array[] __counted_by(len);
        };
```

In this case, the `len` field can only be used to dimension `array` directly, no expressions allowed. This attribute only works for the size of the flexible array itself for now; future work may get it to the point where, for example, it can warn when the allocation size for the structure is not sufficient to hold the array. 

There is some talk of extending this checking to pointer values as well; Apple has [a proposal](<https://discourse.llvm.org/t/rfc-enforcing-bounds-safety-in-c-fbounds-safety/70854>) for a more elaborate `‑fbounds‑safety` flag (for LLVM) implementing this idea. It is a superset of the existing `counted_by()` behavior; it would be more effort to implement and adopt, but will be considered later if it takes off. 

Bounds checking is only useful if the checks are correct, so the existence of false-positive warnings is a problem. Specifically, code that is optimized with [jump threading](<https://en.wikipedia.org/wiki/Jump_threading>) can create false positives. One aspect of this problem has been fixed in GCC 13, while another is still open. This issue is preventing `‑Warray‑bounds` from being enabled by default in kernel builds. There are some ideas circulating for how to mark code where jump threading has been used and suppress the resulting warnings. 

A separate issue entirely is integer-overflow detection. In the C standard, overflow is defined for unsigned integer values, but undefined for signed values and pointers. For the undefined case, GCC provides options to either define the expected behavior or to detect the overflow. There is no option, though, for unsigned overflow, since the behavior is well defined. But unsigned overflow is often unintentional and would be good to detect. Perhaps, she said, there needs to be a new option to allow for detection in this case. 

Back to signed overflow, she noted that the `‑fwrapv` option makes the behavior defined; the variable will wrap around when it overflows. But, while the kernel needs to have overflows trap most of the time, there are occasional spots where it should be allowed. Florian Weimer pointed out that there is a built-in mechanism now that can be used to disable checking for specific operations; Zhao said that she would look into it. 

At this point time ran out, and Zhao was unable to get into the discussion of control-flow integrity options. The picture that came out of the session was clear, though. Quite a bit of work has gone into improving GCC so that it can help in the hardening of the kernel (and other programs too, of course). But, like so many other jobs, the task of defending the kernel against attackers never seems to end. There will be plenty for developers, on both the compiler and kernel sides, to do for the foreseeable future.
