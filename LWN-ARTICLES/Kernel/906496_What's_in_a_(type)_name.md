---
title: "What's in a (type) name?"
url: https://lwn.net/Articles/906496/
date: "September 2, 2022"
category: "Documentation-man pages"
author: "By Jonathan Corbet September 2, 2022"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 2, 2022

The [kernel's manual pages](<https://www.kernel.org/doc/man-pages/>) are in a bit of an interesting position. They are managed as a separate project, distinct from the kernel's documentation, and have the task of documenting both the kernel's system-call interface and the wrappers for that interface provided by the C library. Sometimes the two objectives come into conflict, as can be seen in a discussion that has been playing out over the course of the last year on whether to use C standard type names to describe kernel-defined structures. 

The C `<stdint.h>` header file defines a number of types for developers who need to specify exactly how they need an integer variable to be represented. For example, `int16_t` is a 16-bit, signed type, while `uint64_t` is a 64-bit, unsigned type. This level of control is needed when defining data structures that are implemented by hardware, are exchanged through communications protocols — or are passed between user and kernel space. 

The kernel, though, does not use these types to define its system-call interface. Instead, the kernel has its own types defined internally. Rather than use `uint64_t`, for example, the kernel's API definitions use `__u64`. That has been the situation for a long time — since before the standard C types existed — and is simply part of how the kernel project does things. 

As a general rule, the man pages reflect the kernel's definition of data types. So, for example, [the `bpf()` man page](<https://man7.org/linux/man-pages/man2/bpf.2.html>) defines one piece of the `bpf_attr` union as: 

```
struct {    /* Used by BPF_MAP_*_ELEM and BPF_MAP_GET_NEXT_KEY
                       commands */
    	__u32         map_fd;
    	__aligned_u64 key;
    	union {
    	    __aligned_u64 value;
    	    __aligned_u64 next_key;
    	};
    	__u64         flags;
        };
```

These types are familiar to kernel developers, but they may look a bit strange to user-space developers. Back in April of 2021, man-pages co-maintainer Alejandro Colomar decided to make things look more familiar by rewriting the man pages to use the standard C types instead. Perhaps out of love for a challenge, Colomar [started with the `bpf()` man page](<https://lwn.net/ml/linux-kernel/20210423230609.13519-1-alx.manpages@gmail.com/>); after applying the patch, the above structure was defined as: 

```
struct {    /* Used by BPF_MAP_*_ELEM and BPF_MAP_GET_NEXT_KEY commands */
            uint32_t                     map_fd;
            uint64_t [[gnu::aligned(8)]] key;
            union {
                uint64_t [[gnu::aligned(8)]] value;
                uint64_t [[gnu::aligned(8)]] next_key;
             };
             uint64_t                     flags;
         };
```

This patch was immediately [vetoed](<https://lwn.net/ml/linux-kernel/CAADnVQLf4qe3Hj7cjBUCY4wXb9t2ZjUt=Z=JuygRY0LNNHWAoA@mail.gmail.com/>) by BPF maintainer Alexei Starovoitov, who said: ""The man page should describe the kernel api the way it is in .h file"". Colomar [answered](<https://lwn.net/ml/linux-kernel/56932c68-4992-c5e4-819f-a88f60b3f63a@gmail.com/>) that the actual types used are the same either way, and that his change was better for users: 

> If we have a standard syntax for fixed-width integral types (and for anything, actually), the manual pages should probably follow it, whenever possible. Any deviation from the standard (be it C or POSIX) should have a very good reason to be; otherwise, it only creates confusion. 

Starovoitov [stood firm](<https://lwn.net/ml/linux-kernel/CAADnVQJU=r0qE-4ZHsvX4YndbFgDGvzAgNgVo7kPMGF4jCrVeg@mail.gmail.com/>) in his opposition, though, saying that the man pages should describe the types as they will be defined when code includes the associated kernel header file. 

Colomar [returned](<https://lwn.net/ml/linux-kernel/20210504110519.16097-1-alx.manpages@gmail.com/>) in May 2021 with a new version of the patch that was little changed from its predecessor. Also unchanged was the reception it got. This time, Greg Kroah-Hartman also [expressed his opposition](<https://lwn.net/ml/linux-kernel/YJFxArfp8wN3ILJb@kroah.com/>), saying that the types involved ""are not the same, they live in different namespaces, and worlds, and can not always be swapped out for each other on all arches"". GNU C Library developer Zack Weinberg [disagreed](<https://lwn.net/ml/linux-kernel/CAKCAbMg_eRCsD-HYmryL8XEuZcaM1Qdfp4XD85QKT6To+h3QcQ@mail.gmail.com/>), though: 

> Manpage documentation of C structs is *not* expected to match the actual declaration in the headers. The documented field type is usually assignment-compatible with the actual type, but not always. There's no guarantee whatsoever that the fields are in the same order as the header, or that the listed set of fields is complete. 

This argument failed to convince the kernel community, though, which remained strongly against the change. This discussion then died down for over a year. 

Colomar returned with [a new patch](<https://lwn.net/ml/linux-kernel/20220824185505.56382-1-alx.manpages@gmail.com/>) converting many more files in August 2022; he included the Nacked-by tags he had received from three different developers. Unsurprisingly, those developers had not become more sympathetic toward the idea during the pause. Starovoitov [repeated his opposition](<https://lwn.net/ml/linux-kernel/CAADnVQKiEVL9zRtN4WY2+cTD2b3b3buV8BQb83yQw13pWq4OGQ@mail.gmail.com/>) and asked Colomar to stop sending the patch. 

In response, Colomar [went ahead and applied the patch](<https://lwn.net/ml/linux-kernel/c06008bc-0c13-12f1-df85-3814b74e47f9@gmail.com/>) to the man-pages repository. A kernel patch that had encountered such opposition would almost certainly never have been applied, but the man pages are not a kernel project. Colomar appears to be the only active man-pages maintainer at the moment; longtime maintainer Michael Kerrisk has seemingly vanished from the scene since the [man pages 5.13 release](<https://lwn.net/ml/linux-kernel/11ec92df-0b4c-16f2-a3bc-f55a59ad01ab%40gmail.com/>) in August 2021. So there is nobody who is in a position to overrule Colomar when it comes to decisions in this area. 

Much of the discussion covered the same ground as with the previous versions, but this time Linus Torvalds jumped in as well. He [pointed out](<https://lwn.net/ml/linux-kernel/CAHk-=whfft=qpCiQ=mkaCz+X1MEfGK5hpUWYoM5zWK=2EQMwyw@mail.gmail.com/>) that the kernel's types simply cannot be the same as the standard C types without creating namespace problems: the kernel cannot include `<stdint.h>` to define those types, but also cannot define those types itself in files used by user space without creating conflicts there. Torvalds [agreed with the others](<https://lwn.net/ml/linux-kernel/CAHk-=wgSx8O0=p18C1aQuH4Gw7xmKujBKMEiSitCA7oG2h6WLg@mail.gmail.com/>) that the documentation should match the actual types used. 

> Honestly, I don't think it makes a *huge* amount of difference, but documentation that doesn't actually match the source of the documentation will just confuse somebody in the end. Somebody will go "that's not right", and maybe even change the structure definitions to match the documentation. 

This message, along with [a request from Kroah-Hartman](<https://lwn.net/ml/linux-kernel/YwcPQ987poRYjfoL@kroah.com/>) to revert the change, was enough to convince Colomar to back down. His [concluding words](<https://lwn.net/ml/linux-kernel/ee51a03e-0cc7-06a6-2ae9-e68af02e891f@gmail.com/>) were: 

> You convinced me. The man-pages will document the types exactly as they are in kernel. It's just simpler. 
> 
> As the patch was recently reverted after Greg asked me to do, I'll keep it that way. I guess this closes the man-pages discussion. 

The interesting thing, of course, is that the kernel does, indeed, [define many of the standard types](<https://elixir.bootlin.com/linux/v5.19.5/source/include/linux/types.h#L92>) internally, and there are thousands of variables defined using those types. Using standard C types in the kernel is not, itself, a problem; only using them in the user-space API definitions is. With sufficient will, this might well be a problem that could be overcome, but it would not be a small job. Meanwhile, it seems that the man pages will continue to document the types that are actually used in the kernel's user-space API header files.
