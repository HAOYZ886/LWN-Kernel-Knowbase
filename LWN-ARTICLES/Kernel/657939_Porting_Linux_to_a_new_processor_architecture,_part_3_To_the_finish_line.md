---
title: "Porting Linux to a new processor architecture, part 3: To the finish line"
url: https://lwn.net/Articles/657939/
date: "September 23, 2015"
category: "Architectures-Porting to"
author: "September 23, 2015 This article was contributed by Joël Porquet"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

September 23, 2015

This article was contributed by Joël Porquet

This series of articles provides an overview of the procedure one can follow when porting the Linux kernel to a new processor architecture. [Part 1](<https://lwn.net/Articles/654783/>) and [part 2](<https://lwn.net/Articles/656286/>) focused on the non-code-related groundwork and the early code, from the assembly boot code to the creation of the first kernel thread. Following on from those, the series concludes by looking at the last portion of the procedure. As will be seen, most of the remaining work for launching the `init` process deals with thread and process management.

#### Spawning kernel threads

When `start_kernel()` performs its last function call (to `rest_init()`), the memory-management subsystem is fully operational, the boot processor is running and able to process both exceptions and interrupts, and the system has a notion of time.

While the execution flow has so far been sequential and mono-threaded, the main job handled by `rest_init()` before turning into the boot idle thread is to create two kernel threads: `kernel_init`, which will be discussed in the next section, and `kthreadd`. As one can imagine, creating these kernel threads (and any other kinds of threads for that matter, from user threads within the same process to actual processes) implies the existence of a complex process-management infrastructure. Most of the infrastructure to create a new thread is not architecture-specific: operations such as copying the `task_struct` structure or the credentials, setting up the scheduler, and so on do not usually need any architecture-specific code. However, the process-management code must define a few architecture-specific parts, mainly for setting up the stack for each new thread and for switching between threads.

Linux always avoids creating new resources from scratch, especially new threads. With the exception of the initial thread (the one that has so far been booting the system and that we have implicitly been discussing), the kernel always duplicates an existing thread and modifies the copy to make it into the desired new thread. The same principle applies after thread creation, when the new thread's execution begins for the first time, as it is easier to resume the execution of a thread than to start it from scratch. This mainly means that the newly allocated stack must be initialized such that when switching to the new thread for the first time, the thread looks like it is resuming its execution—as if it had simply been stopped earlier.

To further understand this mechanism, delving a bit into the thread-switching mechanism and more specifically into the switch of execution flow implemented by the architecture-specific context-switching routine `switch_to()` is required. This routine, which is always written in assembly language, is always called by the current (soon to be previous) thread while returning as the next (future current) thread. Part of this trick is achieved by saving the current context in the stack of the current thread, switching stack pointers to use the stack of the next thread, and restoring the saved context from it. As with a typical function, `switch_to()` finally returns to the "calling" function using the instruction address that had been saved on the stack of the newly current thread.

In the case that the next thread had previously been running and was temporarily removed from the processor, returning to the calling function would be a normal event that would eventually lead the thread to resume the execution of its own code. However, for a brand new thread, there would not have been any function to call `switch_to()` in order to save the thread's context. This is why the stack of a new thread must be initialized to pretend that there has been a previous function call, enabling `switch_to()` to return after restoring this new thread. Such a function is usually setup to be a few assembly lines acting as a trampoline to the thread's code.

Note that switching to a kernel thread does not generally involve switching to another page table since the kernel address space, in which all kernel threads run, is defined in every page table structure. For user processes, the switch to their own page table is performed by the architecture-specific routine `switch_mm()`.

#### The first kernel thread

As explained in the [source code](<http://lxr.free-electrons.com/source/init/main.c?v=4.2#L386>), the only reason the kernel thread `kernel_init` is created first is that it must obtain PID 1. This is the PID that the `init` process (i.e. the first user space process born from `kernel_init`) traditionally inherits.

Interestingly, the first task of `kernel_init` is to wait for the second kernel thread, `kthreadd`, to be ready. `kthreadd` is the kernel thread daemon in charge of asynchronously spawning new kernel threads whenever requested. Once `kthreadd` is started, `kernel_init` proceeds with the second phase of booting, which includes a few architecture-specific initializations.

In the case of a multiprocessor system, `kernel_init` begins by starting the other processors before initializing the various subsystems composing the driver model (e.g. devtmpfs, devices, buses, etc.) and, later, using the defined initialization calls to bring up the actual device drivers for the underlying hardware system. Before getting into the "fancy" device drivers (e.g. block device, framebuffer, etc.), it is probably a good idea to focus on having at least an operational terminal (by implementing the corresponding driver if necessary), especially since the early console set up by `early_printk()` is supposed to be replaced by a real, full-featured console shortly after.

It is also through these initialization calls that the initramfs is unpacked and the initial root filesystem (rootfs) is mounted. There are a [few options](<https://www.kernel.org/doc/Documentation/early-userspace/README>) for mounting an initial rootfs but I have found [initramfs](<https://www.kernel.org/doc/Documentation/filesystems/ramfs-rootfs-initramfs.txt>) to be the simplest when porting Linux. Basically this means that the rootfs is statically built at compilation time and integrated into the kernel binary image. After being mounted, the rootfs can give access to the mandatory `/init` and `/dev/console`.

Finally, the init memory is freed (i.e. the memory containing code and data that were used only during the initialization phase and that are no longer needed) and the `init` process that has been found on the rootfs is launched.

#### Executing init

At this point, launching `init` will probably result in an immediate fault when trying to fetch the first instruction. This is because, as with creating threads, being able to execute the `init` process (and actually any user-space application) first involves a bit of groundwork.

The function that needs to be implemented in order to solve the instruction-fetching issue is the page fault handler. Linux is lazy, particularly when it comes to user applications and, by default, does not pre-load the text and data of applications into memory. Instead, it only sets up all of the kernel structures that are strictly required and lets applications fault at their first instruction because the pages containing their text segment have usually not been loaded yet.

This is actually perfectly intentional behavior since it is expected that such a memory fault will be caught and fixed by the page fault handler. This handler can be seen as an intricate switch statement that is able to treat every fault related to memory: from `vmalloc()` faults that necessitate a synchronization with the reference page table to stack expansions in user applications. In this case, the handler will determine that the page fault corresponds to a valid virtual memory area (VMA) of the application and will consequently load the missing page in memory before retrying to run the application.

Once the page fault handler is able to catch memory faults, it is likely that an extremely simple `init` process can be executed. However, it will not be able to do much as it cannot yet request any service from the kernel through system calls, such as printing to the terminal. To this end, the system-call infrastructure must be completed with a few architecture-specific parts. System calls are treated as software interrupts since they are accessed by a user instruction that makes the processor automatically switch to kernel mode, like hardware interrupts do. Besides defining the list of system calls supported by the port, handling system calls involves enhancing the interrupt and exception handler with the additional ability to receive them.

Once there is support for system calls, it should now be possible to execute a "hello world" `init` that is able to open the main console and write a message. But there are still missing pieces in order to have a full-featured `init` that is able to start other applications and communicate with them as well as exchange data with the kernel.

The first step toward this goal concerns the management of signals and, more particularly, signal delivery (either from another process or from the kernel itself). If a process has defined a handler for a specific signal, then this handler must be called whenever the given signal is pending. Such an event occurs when the targeted process is about to get scheduled again. More specifically, this means that when resuming the process, right at the moment of the next transition back to user mode, the execution flow of the process must be altered in order to execute the handler instead. Some space must also be made on the application's stack for the execution of the handler. Once the handler has finished its execution and has returned to the kernel (via a system call that had been previously injected into the handler's context), the context of the process is restored so that it can resume its normal execution.

The second and last step for fully running user-space applications deals with user-space memory access: when the kernel wants to copy data from or to user-space pages. Such an operation can be quite dangerous if, for example, the application gives a bogus pointer, which would potentially result in kernel panics (or security vulnerabilities) if it is not checked properly. To circumvent this problem, it is necessary to write architecture-specific routines that use some assembly magic to register the addresses of all of the instructions performing the actual accesses to the user-space memory in an exception table. As explained in this [LWN article](<http://lwn.net/2001/0222/kernel.php3>) from 2001, ""if ever a fault happens in kernel mode, the fault handler scans through the exception table trying to match the address of the faulting instruction with a table entry. If a match is found, a special error exit is taken, the copy operation fails gracefully, and the system call returns a segmentation fault error.""

#### Conclusion

Once a full-featured `init` process is able to run and give access to a shell, it probably signals the end of the porting process. But it is most likely only the beginning of the adventure, as the port now needs to be maintained (as the internal APIs sometimes change quickly), and can also be enhanced in numerous ways: adding support for multiprocessor and NUMA systems, implementing more device drivers, etc.

By describing the long journey of porting Linux to a new processor architecture, I hope that this series of articles will contribute to remedying the lack of documentation in this area and will help the next brave programmer who one day embarks upon this challenging, but ultimately rewarding, experience.

[The author would like to thank Ena Lupine for her help in writing and publishing these articles.]
