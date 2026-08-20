---
title: "KS2012: Improving tracing and debugging"
url: https://lwn.net/Articles/514898/
date: "September 12, 2012"
category: "Development tools-Kernel tracing"
author: "By Michael Kerrisk September 12, 2012 2012 Kernel Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Michael Kerrisk**  
September 12, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

Day one of the 2012 Kernel Summit saw a discussion on improving kernel tracing and debugging, led by Jason Wessel and Steven Rostedt. Jason's particular interest was how to get better tracing information from users who send in reports for kernel crashes. 

Most of the session focused on Jason's proposal for kernel changes that would allow source line numbers to be displayed as part of the backtrace that is provided in the event of a kernel crash, so as to allow easier diagnosis of the source of the crash. The proposed technique is implemented by including ELF tables with the necessary symbol information in the compiled kernel. With Jason's patches, use of this feature is straightforward: the kernel is configured with `CONFIG_KALLSYMS_LINE_LOCATIONS` enabled and built with debugging information included. Once that is done, then events such as kernel panics will generate a [call trace that includes source file names and line numbers](<https://lwn.net/Articles/515508/>): 

```
Call to panic() with the patch set
        ----------------------------------
        Call Trace:
         [<ffffffff815f3003>] panic+0xbd/0x14 panic.c:111
         [<ffffffff815f31f4>] ? printk+0x68/0xd printk.c:765
         [<ffffffffa0000175>] panic_write+0x25/0x30 [test_panic] test_panic.c:189
         [<ffffffff8118aa96>] proc_file_write+0x76/0x21 generic.c:226
         [<ffffffff8118aa20>] ? __proc_create+0x130/0x21 generic.c:211
         [<ffffffff81185678>] proc_reg_write+0x88/0x21 inode.c:218
         [<ffffffff81125718>] vfs_write+0xc8/0x20 read_write.c:435
         [<ffffffff811258d1>] sys_write+0x51/0x19 read_write.c:457
         [<ffffffff815f84d9>] ia32_do_call+0x13/0xc ia32entry.S:427
```

The improved call-tracing information that is provided by these patches would undoubtedly make life somewhat easier for diagnosing the causes of some kernel crashes. However, there is a cost: the memory footprint of the resulting kernel is much larger. During the session, a figure of 20 MB was mentioned, although in [a mail that he later sent to the kernel summit discussion list](<https://lwn.net/Articles/515507/>), Jason clarified that the figure was more like 10 MB. 

The large increase in kernel memory footprint that results from Jason's technique immediately generated some skepticism on its usefulness. As someone pointed out, such a large increase in kernel size would be unwelcome by users running kernels in cloud-based virtual machines such as Amazon EC2, where the available memory might be limited to (for example) 0.5 GB. Others suggested that it's probably possible to achieve the same result via a suitably built kernel that is loaded by `kexec()` in the event of a kernel crash. (However, there was some questioning of that idea also, since that technique might also carry a significant memory overhead.) 

Linus then weighed in to argue against the proposal altogether. In his view, kernel panics are a small enough part of user bug reports that the cost of this approach is unwarranted; an overhead of something like 1 MB for the increase in memory footprint would be more reasonable, he thought. Linus further opined that one can, with some effort, obtain similar traceback information by loading the kernel into GDB. 

Although Jason's proposed patches provide some helpful debugging functionality, the approach met enough negative response that it seems unlikely to be merged in anything like its current form. However, Jason may not be ready to completely give up on the idea yet. In his mail sent soon after the session, he hypothesized about some modifications to his approach that might bring the memory footprint of his feature down to something on the order of 5MB, as well as other approaches that could be employed so that the end user had greater control over when and if this feature was deployed for a running kernel. Thus, it may be that we'll see this idea reappear in a different form at a later date.
