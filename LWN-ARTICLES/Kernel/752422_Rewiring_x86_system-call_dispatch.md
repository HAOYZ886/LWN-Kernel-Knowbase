---
title: "Rewiring x86 system-call dispatch"
url: https://lwn.net/Articles/752422/
date: "April 23, 2018"
category: System calls
author: "By Jonathan Corbet April 23, 2018"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 23, 2018

Each kernel development cycle includes a vast number of changes that are not intended to change visible behavior and which, as a result, go unnoticed by most users and developers. One such change in 4.17 is a rewiring of how system-call implementations are invoked within the kernel. The change is interesting, though, and provides an opportunity to look at the macro magic that handles system-call definitions. 

In user space, system calls look like ordinary functions, albeit with the strange convention of returning error codes in the global `errno` variable. Those functions are indeed just that, in that they are generally wrapper functions defined in the C library (or some other language-specific implementation). Those wrappers are responsible for organizing the system-call arguments properly (placing them into a set of registers defined by the architecture ABI) and triggering a trap into the kernel, where the real work gets done. 

Imagine that the application is calling `read()`. In the 4.16 kernel, [the implementation of this system call](<https://elixir.bootlin.com/linux/v4.16.3/source/fs/read_write.c#L565>) is: 

```
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
        {
    	struct fd f = fdget_pos(fd);
    	ssize_t ret = -EBADF;
    
    	if (f.file) {
    	    loff_t pos = file_pos_read(f.file);
    	    ret = vfs_read(f.file, buf, count, &pos);
    	    if (ret >= 0)
    		file_pos_write(f.file, pos);
    	    fdput_pos(f);
    	}
    	return ret;
        }
```

The `SYSCALL_DEFINE3()` macro at the beginning declares an implementation for the `read()` system call with three arguments. This is clearly not a normal function definition; one of the first things that jumps out is that the actual declaration of those arguments is done a little strangely. The names of the arguments and their types are separated in this way so that `SYSCALL_DEFINE3()` can perform whatever impedance-matching is required to map those arguments from the system-call ABI into what a normal C function expects. On the x86 architecture, after quite a bit of macro substitution, the `SYSCALL_DEFINE3()` line turns into something like: 

```
asmlinkage long sys_read(int fd, char __user *buf, size_t count) \
            __attribute__((alias(__stringify(SyS_read))));
        asmlinkage long SyS_read(long fd, long buf, long count)
        {
            long ret = SYSC_read((int) fd, (char __user *) buf, (size_t) count);
    	return ret;
        }
        static inline SYSC_read(int fd, char __user *buf, size_t count)
        /* SYSCALL_DEFINE3() expansion ends here, function body follows */
```

As can be seen, two different functions (and one alias) are declared here. `SyS_read()` is declared with the `asmlinkage` attribute (so that it expects all arguments on the stack rather than in registers), and with all arguments declared as having the `long` type, which is how they are passed from user space. This function casts the arguments into the expected types, then calls `SYSC_read()`, which is the name of the function that ends up containing the actual code implementing the system call. Note that it is declared `static inline`, so it will be substituted directly into `SyS_read()`. 

A pointer to the `SyS_read()` version is placed in the appropriate location in the `sys_call_table` array. Then, when the kernel handles a trap for an incoming system call, it comes down to [this bit of code in `do_syscall_64()`](<https://elixir.bootlin.com/linux/v4.16.3/source/arch/x86/entry/common.c#L268>) (again, on x86): 

```
if (likely((nr & __SYSCALL_MASK) < NR_syscalls)) {
    	nr = array_index_nospec(nr & __SYSCALL_MASK, NR_syscalls);
    	regs->ax = sys_call_table[nr](
    		regs->di, regs->si, regs->dx,
    		regs->r10, regs->r8, regs->r9);
        }
```

(The use of `array_index_nospec()` prevents the processor from executing this call speculatively, thus blocking any attempts to create a speculative call to an address outside of `sys_call_table`). Since all of the entries in `sys_call_table` are declared `asmlinkage`, the arguments will be copied from registers onto the stack before the call is made. Note that six registers are pushed onto the stack regardless of the number of arguments that the system call expects; the unneeded values will simply be ignored. This code reflects the maximum of six arguments that any system call may have. 

In 4.17, this mechanism has changed on the x86-64 architecture, thanks to some [work done by Dominik Brodowski](<https://lwn.net/Articles/752426/>). The new convention makes use of the fact that the `pt_regs` structure, created on any trap into the kernel, contains the register state of the user-space process, so it will contain the system-call arguments too. Rather than push all six registers onto the stack, [the relevant line in `do_syscall_64()`](<https://elixir.bootlin.com/linux/v4.17-rc1/source/arch/x86/entry/common.c#L267>) now looks like: 

```
if (likely(nr < NR_syscalls)) {
    	nr = array_index_nospec(nr, NR_syscalls);
    	regs->ax = sys_call_table[nr](regs);
        }
```

The wrapper for the system-call implementation needs to change, since the calling convention has changed. In 4.17, the boilerplate for `read()`, after macro substitution, looks something like: 

```
asmlinkage long __x64_sys_read(const struct pg_regs *regs)
        {
            return __se_sys_read(regs->di, regs->si, regs->dx);
        }
        static long __se_sys_read(long fd, long buf, long count)
        {
            long ret = __do_sys_read((int) fd, (char __user *) buf, (size_t) count);
    	return ret;
        }
        static inline long __do_sys_read(int fd, char __user *buf, size_t count)
```

The `__x64_sys_read()` version goes into `sys_call_table`; its job is to unpack the arguments from the `pt_regs` structure before calling `__se_sys_read()`, which will cast those arguments into the proper type and call the real implementation (now named `__do_sys_read()`). 

Getting to this point required quite a bit of work, including passing over the entire kernel to find and fix every direct call into the old system-call entry points. One might well ask why this effort was made. The new implementation is somewhat cleaner in general, but it also keeps unused, caller-supplied data from ending up on the stack. In current kernels, an attacker could call `read()` with six arguments; the final three would just end up on the stack, unchecked in any way, where they could conceivably turn out to be useful for any of a variety of exploits. By controlling the environment in which system-call code runs a bit more strictly, the new calling convention makes the kernel just a bit harder to compromise.
