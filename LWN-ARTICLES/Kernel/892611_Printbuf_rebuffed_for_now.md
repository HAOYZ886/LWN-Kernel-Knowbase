---
title: Printbuf rebuffed for now
url: https://lwn.net/Articles/892611/
date: "April 28, 2022"
category: Kernel messages; String processing; printk
author: "By Jonathan Corbet April 28, 2022"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
April 28, 2022

There is a long and growing list of options for getting information out of the kernel but, in the real world, print statements still tend to be the tool of choice. The kernel's [`printk()`](<https://www.kernel.org/doc/html/latest/core-api/printk-basics.html?highlight=printk#c.printk>) function often comes up short, despite the fact that it provides a set of kernel-specific features, so there has, for some time, been interest in better APIs for textual output from the kernel. The ["printbuf" proposal](<https://lwn.net/ml/linux-kernel/20220421234837.3629927-1-kent.overstreet@gmail.com/>) from Kent Overstreet is one step in that direction, but will need some changes to make it work well with features the kernel already has. 

A call to `printk()` works well when kernel code needs to output a simple line of text. It is not as convenient when there is a need for complex formatting or when multiple lines of output must be generated. It is possible to use multiple `printk()` calls for even a single line of text, just as it is with `printf()` in user space, but there is a problem: the kernel is a highly concurrent environment, and anything can happen between successive `printk()` calls, including `printk()` calls from other contexts. That results in intermixed output, often described with technical terms like "garbled", that can be painful to make sense of. 

#### Printbuf

An answer to that problem is to assemble the complex output in memory, then to print it in a single operation. That is where printbufs come in. A printbuf is a simple structure containing a pointer to a `char` buffer and some housekeeping information, including the length of that buffer and how much of it contains valid data. Kernel code can set up a printbuf with something like: 

```
#include <linux/printbuf.h>
    
        struct printbuf buf = PRINTBUF;
```

`PRINTBUF` is a simple structure initializer that zeroes the entire thing. There is then a whole set of functions that will append text information to the buffer, including: 

```
void pr_buf(struct printbuf *buf, const char *fmt, ...);
        void pr_char(struct printbuf *buf, char c);
        void pr_newline(struct printbuf *buf);
        void pr_human_readable_u64(struct printbuf *buf, u64 v);
        void pr_human_readable_s64(struct printbuf *buf, s64 v);
        void pr_time(struct printbuf *buf, u64);
        /* ... */
```

`pr_buf()` works like `printk()`, except that the resulting text ends up in `buf` rather than going directly to the system log. Many other functions exist for adding specific types of data to the buffer, some of which is shown above. At any time, the accumulated text can be found in `buf.buf`, which can be passed to `printk()` to output the whole buffer in a single call. When a printbuf is no longer needed, it should be passed to `printbuf_exit()` to free its resources. 

Missing from this discussion so far is any mention of memory management. The printbuf code handles that; it allocates the string buffer, and reallocates it to a larger size whenever it threatens to overflow. Those allocations are done at the `GFP_KERNEL` priority, though printbuf can use `GFP_ATOMIC` if the `atomic` field in the structure is set to a true value. If an allocation fails, the code will make a note of it but will continue, dropping some output but preserving what it can. 

When Overstreet first [posted](<https://lwn.net/ml/linux-kernel/20220419203202.2670193-1-kent.overstreet@gmail.com/>) this code in mid-April, one of the first comments was [a one-liner from Christoph Hellwig](<https://lwn.net/ml/linux-kernel/Yl+T3Mx408HiC6dS@infradead.org/>) asking: ""How does this use case differ from that of lib/seq_buf.c?"" Overstreet, it seems, [was unaware](<https://lwn.net/ml/linux-kernel/20220420051854.qioq4nt4dejpnip6@moria.home.lan/>) of the seq_buf mechanism and, as a consequence, had reimplemented much of it. His response was to propose replacing seq_buf entirely with his new implementation. 

#### Seq_buf

Seq_buf was first [added to the kernel](<https://git.kernel.org/linus/3a161d99c43c>) for the 3.19 release in 2014. It is meant to solve essentially the same problem, though the approach taken is a little different. A seq_buf uses a static buffer allocated by the caller; initialization looks something like this: 

```
#include <linux/seq_buf.h>
    
        char buf[MY_BUFFER_SIZE];
        struct seq_buf seq;
    
        seq_buf_init(&seq, buf, MY_BUFFER_SIZE);
```

The process of generating output in a seq_buf is strikingly similar to the approach used for printbuf; there is a familiar-looking series of functions, including: 

```
int seq_buf_printf(struct seq_buf *s, const char *fmt, ...);
        extern int seq_buf_puts(struct seq_buf *s, const char *str);
        extern int seq_buf_putc(struct seq_buf *s, unsigned char c);
        extern int seq_buf_putmem(struct seq_buf *s, const void *mem,
        			      unsigned int len);
        /* ... */
```

Sending the contents of a seq_buf to the log is a simple matter of calling `printk()` with the previously allocated buffer. This API also includes functions like `seq_buf_to_user()`, which will copy the contents of a seq_buf into user space. On the other hand, it lacks some of the fancier formatting features provided by the printbuf mechanism. Arguably, though, the biggest difference between the two interfaces is the automatic memory management done by printbuf. A seq_buf can run out of space but, in the absence of allocation failures, a printbuf never will. 

#### Reconciling the two

There would appear to be agreement that the printbuf submission brings some useful features, but there is little interest in having two subsystems in the kernel that do the same job. So it is not surprising that Overstreet was [advised](<https://lwn.net/ml/linux-kernel/20220422042017.GA9946@lst.de/>) to set printbuf aside and, instead, add any needed capabilities to seq_buf. Steve Rostedt, who wrote the original seq_buf code, [offered](<https://lwn.net/ml/linux-kernel/20220422113736.460058cc@gandalf.local.home/>) to help with that task. 

Overstreet [was not thrilled](<https://lwn.net/ml/linux-kernel/20220422193015.2rs2wvqwdlczreh3@moria.home.lan/>) with that idea, though: 

> Printbuf is the more evolved, more widely used implementation, and you're asking me to discard it so the kernel can stick with its more primitive, less widely used implementation. 

The "more widely used" claim raised some eyebrows, given that printbuf is not in the kernel and thus, with regard to the mainline, not used at all. He was, it seems, counting uses in his own, out-of-tree, bcachefs code — an argument that tends to carry little weight in the kernel community. 

Meanwhile, [a patch adding printbuf use in the memory-management subsystem](<https://lwn.net/ml/linux-kernel/20220419203202.2670193-4-kent.overstreet@gmail.com/>) drew [questions](<https://lwn.net/ml/linux-kernel/Yl+vHJ3lSLn5ZkWN@dhcp22.suse.cz/>) from Michal Hocko, who was not convinced of the value of the new output that it generates. He later also [raised concerns](<https://lwn.net/ml/linux-kernel/YmKd2iHTHuzaTkE6@dhcp22.suse.cz/>) on the use of dynamic memory allocation for logging from the memory-management subsystem. When trying to log information about, for example, an out-of-memory situation, attempting to allocate more memory tends not to end well; at best it will dip into the final memory reserves that should be dedicated to the task of freeing memory. 

The conversations continued over a few different thread branches, and got somewhat adversarial in a few of them. Overstreet made it clear, with [references](<https://lwn.net/ml/linux-kernel/20220422203057.iscsmurtrmwkpwnq@moria.home.lan/>) to "not-invented-here syndrome" and such, that he was not pleased with the reception given to his code. It began to look like one of those threads that leads to the developer involved walking away from the kernel community altogether. 

Hopefully that is not how this discussion will end, though. The memory-management logging topic will have a session at the upcoming [Linux Storage, Filesystem, and Memory-Management Summit](<https://events.linuxfoundation.org/lsfmm/>). Meanwhile, Overstreet did eventually [come to agree](<https://lwn.net/ml/linux-kernel/20220422215146.i663tn6zzn6blzo3@moria.home.lan/>) that implementing his features on top of the existing seq_buf code might be a viable path forward. Assuming that this direction works out, it could lead to the kind of resolution that the kernel community normally strives for: the incorporation of useful new features without duplicating mechanisms that the kernel already supports. The proof will be in the updated patch sets, if and when they are posted.
