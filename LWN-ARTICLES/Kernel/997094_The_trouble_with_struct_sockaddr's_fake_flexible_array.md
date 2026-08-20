---
title: "The trouble with struct sockaddr's fake flexible array"
url: https://lwn.net/Articles/997094/
date: "November 7, 2024"
category: "Flexible array members; Networking-struct sockaddr"
author: "By Jonathan Corbet November 7, 2024"
---

By **Jonathan Corbet**  
November 7, 2024

Flexible arrays — arrays that are declared as the final member of a structure and which have a size determined at run time — have long drawn the attention of developers seeking to harden the kernel against buffer-overflow vulnerabilities. These arrays have reliably been a source of bugs, so anything that can be done to ensure that operations on them stay within bounds is a welcome improvement. While many improvements, including the recent [counted-by work](<https://lwn.net/Articles/936728/>), have been made, one of the most difficult cases remains. Now, however, developers who are interested in using recent compiler bounds-checking features are trying to get a handle on `struct sockaddr`. 

#### The many faces of `struct sockaddr`

The `sockaddr` structure dates back to the beginning of the BSD socket API; it is used to hold an address corresponding to one side of a network connection. The [4.2 BSD networking implementation notes](<https://digitalassets.lib.berkeley.edu/techreports/ucb/text/CSD-83-146.pdf>) from 1983 give its format as: 

```
struct sockaddr {
            short sa_family;
    	char sa_data[14];
        };
```

The `sa_family` field describes which address family is in use — `AF_INET` for an IPv4 address, for example. `sa_data` holds the actual address, the format of which will vary depending on the family. The implementation notes say that: ""the size of the data field, 14 bytes, was selected based on a study of current address formats"". In other words, 14 bytes — much longer than the four needed for an IPv4 address — should really be enough for anybody. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Need it be said that 14 bytes turned out not to be enough? As new protocols came along, they brought address formats that were longer than would fit in `sa_data`. But the `sockaddr` structure was firmly set in stone as user-space API and could not be changed. It appears in essentially the same form in any modern Unix-like system; on Linux the type of `sa_family` is now `sa_family_t`, but otherwise the structure is the same. 

The result was one of the (many) historic kludges of the Unix API. New protocol implementations typically brought with them a variant of `struct sockaddr` that was suitably sized for the addresses in use; `struct sockaddr_in6` for IPv6 addresses, for example, or `struct sockaddr_ax25` for AX.25. All of the socket API interfaces still specified `struct sockaddr`, but implementations on both sides would use the appropriate structure for the protocol in use. Code on both sides of the API would cast pointers to and from `struct sockaddr` as needed. 

Even now, the documented APIs for system calls like [`connect()`](<https://man7.org/linux/man-pages/man2/connect.2.html>) and library functions like [`getaddrinfo()`](<https://man7.org/linux/man-pages/man3/getaddrinfo.3.html>) use `struct sockaddr`. As a result, both user-space programs and the kernel contain a whole set of casts between that type and the type they are (hopefully) actually using. Needless to say, these casts can be error prone; casting a pointer between different structure types is also deemed to be undefined behavior in current C. But that's the price we pay for API compatibility. 

The advent of IPv6 also brought another type: `struct sockaddr_storage`; it is defined as starting with the same `sa_family` field, but being large enough to hold any of the other `sockaddr` variants. Code dealing with network addresses can allocate a structure of this type and be sure of having enough space to store any address. This structure is now what is often allocated, but it never appears explicitly in the system-call interface. 

#### Making the flexible array explicit

The C language has accumulated a few idioms for the declaration of flexible arrays over the years; specifying a dimension of zero or one are both common (though deprecated) examples. The syntax blessed by the language standard, though, is to omit the dimension entirely: 

```
struct something {
            /* ... */
    	int flex_member[]; /* A flexible array */
        };
```

This syntax makes it clear that a flexible array is in use and that the type declaration cannot be used, on its own, to check for overflows of that array. In no convention is it deemed reasonable to use a dimension of 14 for a flexible array, but that is exactly what now happens with `struct sockaddr`. The actual length of `sa_data` is not known, and has a good chance of being larger than the declared size. It is a flexible array disguised as an ordinary array. 

That usage complicates checking of `struct sockaddr` usage for overflows, but the effects go beyond that; it makes detection of flexible arrays harder across the kernel. As Kees Cook noted in [this 2022 patch](<https://lwn.net/ml/all/20221018095503.never.671-kees@kernel.org/>): 

> One of the worst offenders of "fake flexible arrays" is struct sockaddr, as it is the classic example of why GCC and Clang have been traditionally forced to treat all trailing arrays as fake flexible arrays: in the distant misty past, sa_data became too small, and code started just treating it as a flexible array, even though it was fixed-size. 

As long as this usage remains, the checking tools built into both compilers must treat _any_ trailing array in a structure as if it were flexible; that can disable overflow checking on that array entirely. 

It would be nice to change this usage but, as was noted above, the layout of `struct sockaddr` is wired deeply into the socket interface and cannot be changed without breaking applications. But that doesn't mean that the kernel must treat `sa_data` as anything but a flexible array. To enable that without changing the binary interface, Cook redefined `struct sockaddr` within the kernel to: 

```
struct sockaddr {
            sa_family_t	sa_family;
    	union {
    	    char sa_data_min[14];
    	    DECLARE_FLEX_ARRAY(char, sa_data);
    	};
        };
```

(The [`DECLARE_FLEX_ARRAY()` macro](<https://elixir.bootlin.com/linux/v6.11.6/source/include/linux/stddef.h#L83>) jumps through some hoops needed to declare a flexible array within a union). This change made it clear that `sa_data` is a flexible array, which helped, in turn, in the goal of allowing the compilers to treat trailing arrays as non-flexible unless they are explicitly declared as such. 

This patch was merged for the 6.2 release, and all seemed to be well. But, as Gustavo A. R. Silva pointed out in [this patch series](<https://lwn.net/ml/all/cover.1729802213.git.gustavoars@kernel.org>), there is a problem with this approach. There are many places in the kernel where `struct sockaddr` is embedded within another structure, usually not at the end. That has the result of placing a flexible array in the middle of the embedding structure, which is problematic for fairly obvious reasons; the compiler no longer knows what the offsets to the members after `struct sockaddr` should be. That has resulted in ""thousands of warnings"" when the suitable check is enabled in the compiler. 

Silva's solution was to introduce yet another variant with a familiar form: 

```
struct sockaddr_legacy {
            sa_family_t	sa_family;
    	char		sa_data[14];
        };
```

This structure, which lacks the flexible-array member, was then embedded in the other structures, making the warning go away. Since the embedding cases did not use `sa_data` as a flexible array (otherwise things would never have worked in the first place), this change was deemed safe to make. 

Networking maintainer Jakub Kicinski [was not convinced](<https://lwn.net/ml/all/20241031180145.01e14e38@kernel.org/>) about this change, though. He suggested that perhaps Cook's patch should be reverted instead, and a new type should be added for places where a flexible array is actually needed. Cook [acknowledged](<https://lwn.net/ml/all/202411031920.BEF6CEBCD@keescook/>) this suggestion as ""a pretty reasonable position"" and started to ponder on alternatives. He concluded: ""Now, if we want to get to a place with the least ambiguity, we need to abolish sockaddr from the kernel internally, and I think that might take a while."" 

#### Leaving `struct sockaddr` behind

In early November, Cook returned with [a brief patch series](<https://lwn.net/ml/all/20241104221450.work.053-kees@kernel.org>) meant to show what that approach would look like. It begins by reverting the 2022 patch, returning `struct sockaddr` to its original non-flexible form. There is [a patch](<https://lwn.net/ml/all/20241104222513.3469025-2-kees@kernel.org>) adding comments to places in the networking code that are known to use this structure within its original bounds; they do not need to be changed, and do not need `sa_data` to be flexible. But that still leaves many uses of `struct sockaddr` where the data area may, in reality, be larger than 14 bytes. 

The solution for many of those places is just to use `struct sockaddr_storage` instead. Indeed, parts of the network stack already use that structure, but then cast pointers to `struct sockaddr` for functions that expect that type. One example is [`inet_addr_is_any()`](<https://elixir.bootlin.com/linux/v6.11.6/source/net/core/utils.c#L402>), which takes a `struct sockaddr *` argument, but is only called by functions using `struct sockaddr_storage`. In this case, [the solution](<https://lwn.net/ml/all/20241104222513.3469025-4-kees@kernel.org>) is to change the prototype of the function to match what is really being passed to it and remove the casts from the callers. 

Some changes will require more churn, even if they are conceptually simple. The `getname()` callback (in the [`proto_ops` structure](<https://elixir.bootlin.com/linux/v6.11.6/source/include/linux/net.h#L162>)) has long expected a pointer to a `sockaddr_storage` structure, but its prototype was never changed to match. The [patch](<https://lwn.net/ml/all/20241104222513.3469025-5-kees@kernel.org>) eliminating the use of `struct sockaddr` for `getname()` mostly consists of name changes and cast removal, but it touches 66 files. It also, as Cook noted in the cover letter, is still lying to the compiler in cases where the backing structure is actually smaller than `struct sockaddr_storage`, ""these remain just as safe as they used to be. :)"" 

This series shows that truly eliminating the use of this structure's `sa_data` field as a flexible array in disguise will involve a fair amount of work and code churn. Even so, Kicinski [commented](<https://lwn.net/ml/all/20241105171607.48c0c24d@kernel.org>) that it ""feels like the right direction"". So, while `struct sockaddr` will likely remain part of the kernel's system-call API forever, its use within the kernel can be expected to fade away over time. A design miscalculation made over 40 years ago may finally stop impeding the use of modern memory-safety tools.
