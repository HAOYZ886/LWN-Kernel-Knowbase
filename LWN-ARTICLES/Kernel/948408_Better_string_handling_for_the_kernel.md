---
title: Better string handling for the kernel
url: https://lwn.net/Articles/948408/
date: "October 26, 2023"
category: String processing
author: "By Jonathan Corbet October 26, 2023"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 26, 2023

The C programming language is replete with features that seemed like a good idea at the time (and perhaps even were good ideas then) that have not aged well. Most would likely agree that string handling, and the use of NUL-terminated strings, is one of those. Kernel developers have, for years, tried to improve the handling of strings in an attempt to slow the flow of bugs and vulnerabilities that result from mistakes in that area. Now there is an early discussion on the idea of moving away from NUL-terminated strings in much of the kernel. 

The biggest problem with NUL-terminated strings is that they carry no information about the size of the buffer that contains them; that makes it easy to create buffer overflows. Over the years, a number of APIs have been created in an attempt to address this problem, adding functions like [`strncpy()`](<https://man7.org/linux/man-pages/man3/strncpy.3p.html>), [`strlcpy()`](<https://lwn.net/Articles/612244/>), and [`strscpy()`](<https://lwn.net/Articles/643376/>), each of which is claimed to be better than its predecessors. None of them change the core concept of a NUL-terminated string, though, and none of them pleases everybody. 

Over the years, the kernel has seen a long series of patches exchanging calls from one set of string functions for another. Sometimes those conversions have introduced bugs of their own, and maintainers have not always been happy with this work. Recently, in response to [a patch](<https://lwn.net/ml/linux-kernel/20231018-strncpy-drivers-nvme-host-fabrics-c-v1-1-b6677df40a35@google.com/>) converting `strncpy()` calls to `strscpy()` in the NVMe driver subsystem, Christoph Hellwig [questioned the value](<https://lwn.net/ml/ksummit-discuss/20231019054642.GF14346@lst.de/>) of these changes: 

> We need to stop pretending that direct manipulation of nul-terminated strings is a good idea. I suspect the churn of replacing one helper with another, maybe slightly better, one probably introduces more bugs than it fixes. 

Truly solving the problem, he said, requires using something better than NUL-terminated strings. He described the "seq_buf" API as ""a good start"" in this direction. 

Seq_buf was [introduced in 2014](<https://git.kernel.org/linus/3a161d99c43c>) for the 3.19 kernel release as an improved way to build strings in the tracing subsystem. It is organized around this structure: 

```
struct seq_buf {
    	char			*buffer;
    	size_t			size;
    	size_t			len;
    	loff_t			readpos;
        };
```

There is nothing revolutionary here; `buffer` points to the memory allocated by the string, and `size` is the size of that buffer. The length of the string stored in `buffer` is kept in `len`; the `readpos` field is used for reading through a string. The user must allocate `buffer` separately and attach it to a `seq_buf` with a call to: 

```
void seq_buf_init(struct seq_buf *s, char *buf, unsigned int size);
```

There is a whole set of functions for working with this structure, most of which have a fairly obvious purpose: 

```
void seq_buf_clear(struct seq_buf *s);
        int seq_buf_printf(struct seq_buf *s, const char *fmt, ...);
        int seq_buf_puts(struct seq_buf *s, const char *str);
        /* ... */
```

There are quite a few more; see [`include/linux/seq_buf.h`](<https://elixir.bootlin.com/linux/v6.6-rc6/source/include/linux/seq_buf.h>) for the full list. 

Operations on a seq_buf are not allowed to overflow the given buffer; should an attempt be made to store too much data, a note will be made of the fact (`len` is set to a value greater than `size`) and future operations will fail. A call to `seq_buf_has_overflowed()` will return a boolean telling whether a seq_buf is in a good condition or not. This API allows for a long series of string operations, with only a single `seq_buf_has_overflowed()` check needed at the end to ensure that all is well; that simplifies error-handling code significantly. 

There is no accessor function to obtain the string itself; users can just reach into the structure and use the `buffer` field directly. That string is maintained NUL-terminated, so it can be passed to functions expecting such strings — though they should not modify the buffer directly, of course. 

The `readpos` field is only used within the tracing code. As a way of reducing the memory used by `struct seq_buf` in general, Matthew Wilcox recently posted [a patch](<https://lwn.net/ml/linux-kernel/20231019194514.2115506-2-willy@infradead.org/>) removing that field and pushing the concept of a read position into the code that uses it. That patch [seems likely](<https://lwn.net/ml/linux-kernel/20231019171439.1c50a16e@gandalf.local.home/>) to be accepted, so `readpos` will probably not be a part of this structure going forward. 

Kent Overstreet recently posted an alternative to seq_buf, called "[printbuf](<https://lwn.net/ml/linux-kernel/20220421234837.3629927-1-kent.overstreet@gmail.com/>)", that takes a somewhat different approach. In the discussion of that patch set, he was [advised](<https://lwn.net/Articles/892611/>) to improve seq_buf instead, a suggestion that he has declined to pursue. So printbuf remains outside of the kernel, but it does appear in linux-next as part of the bcachefs series that (presumably) will be merged for 6.7. Whether printbuf will then be seen as an alternative to seq_buf is unclear, but the community is unlikely to want to maintain two similar string abstractions in the long term. 

Will the kernel switch over to large-scale use of seq_buf (or printbuf) in the long term? It is hard to tell how far a conversion might go. Kees Cook [said](<https://lwn.net/ml/ksummit-discuss/202310182248.9E197FFD5@keescook/>) that, while seq_buf ""shines"" for complex string manipulation, it is more trouble than it is worth for the simple cases, which can be easily verified as correct in code using simpler APIs. Willy Tarreau, though, [said](<https://lwn.net/ml/ksummit-discuss/ZTDUYSBS7AO9pAgm@1wt.eu/>) that strings with explicit length counters are easier to work with in general, lead to simpler code, and can yield better performance as well. Meanwhile, Linus Torvalds, [seemed mostly interested](<https://lwn.net/ml/ksummit-discuss/CAHk-=wj4BZei4JTiX9qsAwk8PEKnPrvkG5FU0i_HNkcDpy7NGQ@mail.gmail.com/>) in getting rid of the remaining `strlcpy()` calls in the kernel and has not expressed an opinion on changing the kernel's handling of strings in general. 

This conversation might encourage some developers to start using seq_buf in place of NUL-terminated C strings. Such a move is likely to start relatively slowly, though. If, over time, this approach starts to show benefits, there may eventually be a push to do conversions on a wider scale. But, no matter how it goes, the kernel seems likely to be managing a lot of NUL-terminated strings for a long time yet.
