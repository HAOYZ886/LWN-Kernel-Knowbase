---
title: Safer flexible arrays for the kernel
url: https://lwn.net/Articles/908817/
date: "September 22, 2022"
category: Flexible array members
author: "By Jake Edge September 22, 2022 LSS EU"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
September 22, 2022

* * *

[LSS EU](<https://lwn.net/Archives/ConferenceByYear/#2022-Linux_Security_Summit_Europe>)

At the 2022 [Linux Security Summit Europe](<https://events.linuxfoundation.org/linux-security-summit-europe/>) (LSS EU), Gustavo A. R. Silva reported in on work he has been doing on "flexible" arrays in the kernel. While these arrays provide some ... flexibility ... they are also a source of bugs, which can often result in security vulnerabilities. He has been working on ways to make the use of flexible arrays safer in the kernel. 

Silva has a background in embedded systems, working with both realtime operating systems (RTOS) and embedded Linux. For the last six years, he has has been working as an upstream kernel engineer. He collaborates with the [Kernel Self Protection Project](<https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project>) (KSPP) and the Linux kernel division of the Google open-source security team. 

#### Trailing and flexible arrays

He began with an introduction to C arrays, starting with the simplest, `int happy_array[10];`, which declares an array holding ten elements of type `int` that can be indexed using the values zero to nine. `happy_array` is "going to remain happy as long as we access it within its boundaries". But C does not enforce those limits, so developers must do so; if they do not, they end up in what he likes to call "The Land of Possibilities", also known as undefined behavior. 

A "trailing" array is one that is declared as the last field in a structure. They can have a concrete size, as with `happy_array`, or they can represent a "blob" of data that is tacked onto the structure at run time. For example: 

```
struct blob_holder {
            ...
            size_t count;
            unsigned char blob[];
        }
```

Typically, some element in the structure holds the length of the blob, such as `count` here. In this way, trailing arrays are used to build variable-length objects (VLOs) in C. So a flexible array is simply a trailing array used as a VLO; its size is determined at run time. A flexible structure is a `struct` that contains a flexible array as its last element. There are three ways to declare flexible arrays, Silva said. Two of those are designated as "fake" flexible arrays because they do not use the C99 empty-bracket declaration (i.e. "true" flexible arrays) as above. Many of those fake uses predate C99 entirely and they declare either zero- or one-element arrays to use as flexible arrays. This usage leads to bugs. 

Declaring a one-element flexible array is a "buggy hack". The problem is that the single element is counted toward the size of the array (and enclosing structure), which can easily lead to off-by-one errors. The count field in the structure is one larger than what should be allocated, so `count - 1` needs to be used consistently. When analyzing existing code that uses flexible structures of that sort, one must always consider the uses of `sizeof()` for the array and structure. Often that analysis will find existing off-by-one and other bugs in the code. 

A zero-element fake flexible array is a GNU extension that was added to work around the lack of true flexible arrays in the language at that time. They are somewhat less buggy than the one-element arrays, since they do not contribute to the size of the enclosing structure. True flexible arrays must appear last in the structure, which is enforced by the compiler. Either of the fake flexible array variants can appear anywhere in a structure, though, which can lead to other kinds of problems, of course. 

#### Problems

The `sizeof()` operator returns different values for the three variants. For the one-element variant, the array's size is that of one element of the type of the array; it is zero for the zero-element variant. But for true flexible arrays, `sizeof()` gives a compile-time error because the size is not known. 

[ ![\[Gustavo A. R. Silva\]](https://static.lwn.net/images/2022/lsseu-silva-sm.png) ](<https://lwn.net/Articles/909132/>)

The first flexible-array-transformation [fix](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=76497732932f>) that he did as part of his KSPP work shows the kind of problem that can stem from fake flexible arrays. A zero-length array was declared at the end of a structure, but later someone [added a field for read-copy-update](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e48f129c2f20>) (RCU) _after_ the flexible array. The compiler did not complain, so the bug persisted from 2011 until he fixed it in 2019. He used a true flexible array declaration (and moved it to the end); now if someone adds a new structure member at the end, the compiler will report an error. 

There has been an effort to enable array-bounds checking in the compiler with the `-Warray-bounds` option, but the fake flexible arrays were causing too many false-positives (along with finding some real bugs). It is not uncommon for a flexible array to be indexed directly with a value that is beyond the "end" of the array. Those needed to be fixed before bounds checking could be turned on. 

He [fixed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c1e4726f465440>) a simple example in mid-2021. A one-element array was being accessed with `[1]`, which is obviously one element too far; changing it to a true flexible array got rid of the warning. Others are a bit [more elaborate](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=39107e8577ad>), but boil down to a switch to a true flexible array; removing the one-element array also got rid of a few `count - 1` calculations for allocation sizes. 

Flexible arrays can be the source or target of `memcpy()` operations and it would be nice to have them participate in the [hardened `memcpy()` effort](<https://lwn.net/Articles/864521/>). When `CONFIG_FORTIFY_SOURCE` is enabled for the kernel, `memcpy()` uses the [`__builtin_object_size()` function](<https://gcc.gnu.org/onlinedocs/gcc/Object-Size-Checking.html>) (with a `type` argument of 1) to calculate the sizes of the source and destination at run time. 

For true flexible arrays, though, that function returns -1 because it cannot determine the size. Fake flexible arrays _do_ have a size, but it turns out that `__builtin_object_size()` still returns -1 for those. Combining that with the behavior of `sizeof()` makes things all a bit confusing as he showed in his [slides](<https://static.sched.com/hosted_files/lsseu2022/5e/lsseu2022_flex_array_transformations.pdf>): 

```
__builtin_object_size(flex_struct->one_element_array, 1) == -1
        __builtin_object_size(flex_struct->zero_length_array, 1) == -1
        __builtin_object_size(flex_struct->flex_array_member, 1) == -1
    
        sizeof(flex_struct->one_element_array) == size-of-element-type
        sizeof(flex_struct->zero_length_array) == 0
        sizeof(flex_struct->flex_array_member) == ? /* Error */
```

Because `__builtin_object_size()` cannot determine a size for trailing arrays, no bounds checking is done in `memcpy()` (with `CONFIG_FORTIFY_SOURCE`) for those arrays today. What's even stranger, perhaps, is that `__builtin_object_size()` returns -1 for _any_ trailing array, even if it has a specified size greater than one. Because `__builtin_object_size()` does not return a size for trailing arrays, even those that it ostensibly could determine, no bounds checking is done in `memcpy()` (with `CONFIG_FORTIFY_SOURCE`) for those arrays today. The reason for this `__builtin_object_size()` behavior is legacy code that declares trailing arrays with a fixed length—but treats them as flexible arrays. He showed a BSD version of `struct sockaddr` with a trailing array, `char sa_data[14]`, that can actually hold up to 255 bytes at run time. 

In order to allow `memcpy()` to do sanity checking on trailing arrays, this ambiguity in declarations for flexible arrays needs to be eliminated. All arrays that are meant to be used as flexible arrays should be declared as true flexible arrays using `[]`; then, compilers can be instructed to treat fixed-length trailing arrays as regular fixed-length arrays. He pointed to a [GCC bug report](<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=101836>) for a compiler change to address the problem. 

#### Compiler flag

There is a new compiler flag in the upcoming GCC 13 and Clang 16 releases that will allow developers to set the level of strictness for flexible arrays: `‑fstrict‑flex‑arrays[=n]` (often abbreviated as `-fsfa`). The default setting for `n` is 0, which means no change from today and all trailing arrays are treated as flexible by `__builtin_object_size()`. Values of `n` from 1 to 3 increase the strictness of enforcement by changing the behavior of `__builtin_object_size()`: 

  * **`-fsfa=1`** : only trailing arrays that are declared with `[1]`, `[0]`, and `[]` are treated as flexible arrays; `__builtin_object_size()` returns the proper length for others. 
  * **`-fsfa=2`** : only trailing arrays that are declared with `[0]` and `[]` are treated as flexible arrays; `__builtin_object_size()` returns the proper length for others. 
  * **`-fsfa=3`** : only trailing arrays that are declared with `[]` are treated as flexible arrays; `__builtin_object_size()` returns the proper length for any with a concrete size. 

Unfortunately, the Clang developers have not (yet?) been convinced to add `-fsfa=3`; there is an ongoing discussion about it, Silva said. The work to transform the flexible arrays in the kernel to true flexible arrays had been going on for several years and there is still more to do. Transforming uses of zero-element arrays is fairly straightforward, but one-element arrays are more difficult to transform because they require more code inspection to look for off-by-one problems. Once that is done, and the compilers are available, `memcpy()` will be able to bounds-check all trailing (non-flexible) arrays, so all arrays of fixed size will finally be bounds-checked in the kernel. 

So there is a path toward getting all of those arrays bounds-checked, what about checking for actual flexible arrays? It is a more challenging case, Silva said, but there are proposals for ways to handle it. The key is to identify the structure member that holds the length of the array. That could be done with an attribute on the array like the following: 

```
struct bounded_flex_struct {
    	...
    	size_t elements;
    	struct foo flex_array[]
    	  __attribute__((__element_count__(elements)));
        };
```

There are some user-space API issues to work out, however, when switching from one-element flexible arrays to true flexible arrays. The first attempt at supporting both the existing API and the new way of doing things duplicated the fields in user-facing structures and placed them inside a `union` so that user space could use the array one way and the kernel could use it the other: 

```
struct farray {
            union {
                struct {
                    ... /* renamed versions of the members */
                    size_t renamed_count;
                    int orig_array_name[1];
                };
                struct {
                    ... /* members with existing names */
                    size_t count;
                    int orig_array_name_flex[];
                };
            };
        };
```

Doing that caused a lot of code churn, so the `__DECLARE_FLEX_ARRAY()` helper macro was added that would go in a union that just contained the arrays: 

```
struct farray {
            ...
            size_t count;
            union {
                int orig_array_name[1];
                __DECLARE_FLEX_ARRAY(int, orig_array_name_flex);
            };
        };
```

In both cases, user space will continue to use `orig_array_name`, while the kernel will use `orig_array_name_flex`. One thing to note is that the size of the structure does not change; the one-element array will still contribute to the size of the structure. 

#### Status, conclusions, and questions

At this point, most of the zero-length arrays in the kernel have been transformed, including handling any user-space API issues. But there is nothing stopping new ones from being added, so he asked kernel developers not to introduce new uses. Transformations for one-element arrays are still a work in progress; that work is more challenging and there is a need to ensure that the maintainers of the code being changed feel comfortable that the changes have not broken anything. To that end, he is using a variety of `diff`-like tools to try to verify that no significant changes have been made by the transformation process. 

It is important to turn all of the kernel uses of flexible arrays into true flexible arrays—and then to ensure that no new uses of zero- or one-length flexible arrays are added, he reiterated. The security of the kernel can be significantly improved with `‑fstrict‑flex‑arrays=3`, which means it is important to convince the Clang developers to support that setting. This work has already found vulnerabilities in the kernel and will surely find more as it progresses. It is going to take some time but there is a clear vision of how we get to the point where all trailing arrays, fixed-size or flexible, will be bounds-checked in `memcpy()`. 

Silva took a few comments and questions at the end of the talk. LSS EU organizer Elena Reshetova noted that when the conversion from `atomic_t` to `recount_t` was done, those developers faced a similar problem with stopping developers from adding new uses of the types they were trying to convert. They ended up integrating a test into the [0-day test robot](<https://01.org/lkp/documentation/0-day-brief-introduction>) to catch those introductions and send email. That worked well and she encouraged Silva to try something like that. 

I asked what reasons the Clang developers had for opposing the strictest setting on the new compiler flag. Silva said that their position is "just don't use zero-length arrays" but he deferred to Kees Cook, who said he could speak to the "minutiae of that". The Clang folks point out that zero-length arrays are not legal C, according to the standard, so if the GNU extension allowing them is removed, zero-length arrays do not exist so the `=2` level is sufficient. Adding another option to support having zero-length arrays that are not flexible arrays seems pointless within that community, Cook said. 

"Unfortunately, that's not the reality of our world." When the GNU extension was added, some code used zero-length arrays as flexible arrays, while other code used them as actual arrays with no elements in order to place markers inside structures, for example. In addition, there are arrays in the kernel that typically have some fixed size but that size may fall to zero in certain configurations. 

There are probably ways to work around the lack of that option for Clang, Cook said, but it would be much easier for the Clang developers to accept the reality that zero-length arrays exist and that the kernel (at least) wants to be able to stop treating them as flexible arrays. There is a flag available to warn on the use of zero-length arrays, but it produces 60,000 warnings on the kernel code, so that is not a sensible path either, Silva said. It is clear that the hope is for the Clang folks to relent on this particular point. 

[I would like to thank LWN subscribers for supporting my travel to Dublin for the Linux Security Summit Europe.]
