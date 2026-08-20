---
title: A struct sockaddr sequel
url: https://lwn.net/Articles/1045453/
date: "November 14, 2025"
category: "Flexible array members; Networking-struct sockaddr; Releases-6.19"
author: "By Jonathan Corbet November 14, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 14, 2025

One of the many objectives of the [Linux Kernel Self-Protection Project (KSPP)](<https://kspp.github.io/>), which just [completed ten years of work](<https://hachyderm.io/@kees/115500589609430241>), is to ensure that all array references can be bounds-checked, even in the case of flexible array members, the size of which is not known at compile time. One of the most challenging flexible array members in the kernel is not even declared as such. Almost exactly one year ago, LWN [looked at](<https://lwn.net/Articles/997094/>) the effort to increase safety around the networking subsystem's heavily used `sockaddr` structure. One year later, Kees Cook is still looking for a way to bring this work to a close. 

In short, the problem is that `struct sockaddr` is traditionally defined as: 

```
struct sockaddr {
            short sa_family;
    	char sa_data[14];
        };
```

The `sa_data` field was more than large enough to hold a network address in the early 1980s when this structure was first defined for BSD Unix, but it is not large enough now. As a result, a great deal of code, in both the kernel and user space, passes around `struct sockaddr` pointers that, in truth, point to different structures with more space for the addresses they need to hold. In other words, `sa_data` is being treated as a flexible array member, even though it is not declared as one. The prevalence of `struct sockaddr` has thrown a spanner into the works of many attempts to better check the uses of array members in structures. 

At the end of last year's episode, much of the kernel had been changed to use `struct sockaddr_storage` (actually implemented as [`struct __kernel_sockaddr_storage`](<https://elixir.bootlin.com/linux/v6.17.7/source/include/uapi/linux/socket.h#L16>)), which has a data array large enough to hold any known network address. An attempt was made to change the definition `struct sockaddr` to make its `sa_data` field into an explicit flexible array member, but that work ran into a minor snag. There are many places in the kernel where `struct sockaddr` is embedded within another structure. In most of these cases, `sa_data` is _not_ treated as a flexible array member, so developers have freely embedded `struct sockaddr` anywhere within the containing structure, often not at the end. 

If `sa_data` is redefined as a flexible array member, the compiler no longer knows how large the structure will actually be. That, in turn, means that the compiler does not know how to lay out a structure containing `struct sockaddr`, so it guesses and emits a warning. Or, in the case of a kernel build, tens of thousands of warnings. Kernel developers, as it turns out, would rather face the prospect of an array overflow than a warning flood of that magnitude, so this work came to a halt. 

One possible solution would be to replace embedded `struct sockaddr` fields with `struct sockaddr_storage`, eliminating the flexible array member. But that would bloat the containing structures with memory that is not needed, so that approach is not popular either. 

Instead, Cook is working on [a patch series](<https://lwn.net/ml/all/20251104002608.do.383-kees@kernel.org>) that introduces yet another `struct sockaddr` variant: 

```
struct sockaddr_unsized {
    	__kernel_sa_family_t	sa_family;	/* address family, AF_xxx */
    	char			sa_data[];	/* flexible address data */
        };
```

Its purpose is to be used in internal network-subsystem interfaces where the size of `sa_data` needs to be flexible, but where its actual size is also known. For example, the `bind()` method in [`struct proto_ops`](<https://elixir.bootlin.com/linux/v6.17.7/source/include/linux/net.h#L161>) is defined as: 

```
int	(*bind) (struct socket *sock,
    		 struct sockaddr *myaddr,
    		 int sockaddr_len);
```

The type of `myaddr` can be changed to `struct sockaddr_unsized *` since `sockaddr_len` gives the real size of the `sa_data` array. Cook's patch series does many such replacements, eliminating the use of variably sized `sockaddr` structures in the networking subsystem. With that done, there are no more uses of `struct sockaddr` that read beyond the 14-byte `sa_data` array. As a result, `struct sockaddr` can be reverted to its classic, non-flexible definition, and array bounds checking can be applied to code using that structure. 

That change is enough to make all of those warnings go away, so many would likely see it as a good stopping point. There is still, though, the matter of all those `sockaddr_unsized` structures, any of which might be the source of a catastrophic overflow at some point. So, once the dust settles from this work, we are likely to see some attention paid to implementing bounds checking for those structures. One possible approach mentioned in the patch set is to eventually add an `sa_data_len` field, so that the structure would contain the length of its `sa_data` array. That would make it easy to document the relationship between the fields with the [`counted_by()` annotation](<https://lwn.net/Articles/936728/>), enabling the compiler to insert bounds checks. 

While the ability to write new code in Rust holds promise for reducing the number of memory-safety bugs introduced into the kernel, the simple fact is that the kernel contains a huge amount of C code that will not be going away anytime soon. Anything that can be done to make that code safer is thus welcome. The many variations of `struct sockaddr` that have made the rounds may seem silly to some, but they are a part of the process of bringing a bit of safety to an API that was defined over 40 years ago. Ten years of KSPP have made the kernel safer, but the job is far from done.
