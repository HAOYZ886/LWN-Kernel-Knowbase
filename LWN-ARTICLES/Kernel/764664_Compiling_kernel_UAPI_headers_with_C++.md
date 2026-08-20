---
title: Compiling kernel UAPI headers with C++
url: https://lwn.net/Articles/764664/
date: "September 13, 2018"
category: Build system
author: "By Jonathan Corbet September 13, 2018"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 13, 2018

Linux kernel developers tend to take a dim view of the C++ language; it is seen, rightly or wrongly, as a sort of combination of the worst (from a system-programming point of view) features of higher-level languages and the worst aspects of C. So it takes a relatively brave person to dare to discuss that language on the kernel mailing lists. David Howells must certainly be one of those; he not only brought up the subject, but is working to make the kernel's user-space API (UAPI) header files compatible with C++. 

If somebody were to ask why this goal is desirable, they [would not be the first](<https://lwn.net/ml/linux-api/20180905165552.GB25206@kroah.com/>) to do so. The question has not actually gotten a complete answer, but some possible motivations come to mind. The most obvious one is that some developers might actually want to write programs in C++ that need access to the kernel's API; there is no accounting for taste, after all. For most system calls, the details of the real kernel API (as opposed to the POSIX-like API exposed by the C library) tend to be hidden, but there are exceptions; the most widespread of those is almost certainly the [`ioctl()`](<http://man7.org/linux/man-pages/man2/ioctl.2.html>) system call. There is a large set of structures used with `ioctl()`; their definition is a big part of the kernel's UAPI. If a C++ compiler cannot compile those UAPI definitions, then those `ioctl()` calls cannot be invoked from C++. 

C++ got its start as a sort of superset of C, so most C code could, in the early days, be compiled with a C++ compiler. The two languages have diverged over the years, though, making it easier to write C code that can no longer be compiled in that way. A look at the changes in [Howells's patch set](<https://lwn.net/ml/linux-api/153616286704.23468.584491117180383924.stgit@warthog.procyon.org.uk/>) gives some good examples of where things can go wrong. 

One common stumbling point is the use of identifiers that C++ has claimed as keywords. The [`drm_i810_dma` structure](<https://elixir.bootlin.com/linux/latest/source/include/uapi/drm/i810_drm.h#L268>), for example, contains a member called `virtual`, while [`struct virtio_net_ctrl_header`](<https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/virtio_net.h#L144>) has a member called `class`. Given the frequent use of members called `private` in the kernel, it is surprising that only one ([`struct keyctl_dh_params`](<https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/keyctl.h#L66>)) seems to have made it into the UAPI. The C++ compiler gets rather grumpy when it encounters those keywords used as identifiers, so something needs to change if the UAPI headers are to be acceptable to it. 

One developer [suggested](<https://lwn.net/ml/linux-api/20180905125636-mutt-send-email-mst@kernel.org/>) that, for example, C++ developers could be asked to compile their programs with a command-line option like `-Dclass=_class` to sidestep the problem. It turns out, though, that this approach, while it is indeed effective at getting the structure in question to compile under C++, has a certain risk of creating unintended difficulties elsewhere in the program. So a different approach is necessary. The solution that was chosen is to change the definition of the structure to look like this: 

```
struct virtio_net_ctrl_hdr {
    	union {
        #ifndef __cplusplus
    	    __u8 class;
        #endif
    	    __u8 _class;
    	};
    	__u8 cmd;
        };
```

The addition of the anonymous union allows the old (C++ keyword) name to be used in C code, while also allowing the addition of a new name that can be used under either language. Changing the structures in this way was [not universally popular](<https://lwn.net/ml/linux-api/20180905165436.GA25206@kroah.com/>), but there do not appear to be a lot of good alternatives, given that breaking code written in C is not acceptable. 

There are various other problems to be solved; for example, ending a structure with an array of unspecified length is not allowed in C++. So a definition like the rather tersely named `[struct bkey](<https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/bcache.h#L21>)`: 

```
struct bkey {
    	__u64	high;
    	__u64	low;
    	__u64	ptr[];
        };
```

must be changed by giving `ptr` an explicit dimension of zero. Other problems turn out to be simply bugs; some structures were defined using kernel-specific types that are not available in user space, for example. In at least [one case](<https://lwn.net/ml/linux-api/153616291029.23468.16421004714304578585.stgit@warthog.procyon.org.uk/>), the structure involved should never have been exposed to user space to begin with and had never been used in communications with the kernel. Cleaning such things up makes sense even if one does not care about the larger goal of C++ compatibility. 

The [final step](<https://lwn.net/ml/linux-api/153616295066.23468.6133013878008690375.stgit@warthog.procyon.org.uk/>) in the patch series is the addition of a script that will feed (almost) all of the UAPI header files to `g++` as part of the kernel's build process. The output of this compilation is discarded, but it serves a useful purpose; any developer who breaks the ability to compile those files under C++ will get some immediate feedback to that effect. At least, they will if they have `g++` installed; otherwise the test is skipped to avoid breaking the kernel build as a whole. Should this series be merged, kernel developers will not necessarily like C++ any more than they do now, but they will at least be more friendly toward C++ developers trying to use their exported API headers.
