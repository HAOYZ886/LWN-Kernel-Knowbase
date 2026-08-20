---
title: A nasty FPU bug
url: https://lwn.net/Articles/89586/
date: "June 15, 2004"
category: "Security-Vulnerabilities"
author: 
---

The problem was initially [reported as a gcc bug](<http://gcc.gnu.org/bugzilla/show_bug.cgi?id=15905>). If you execute this code: 

```
static void Handler(int ignore)
        {
    	char fpubuf[108];
    	__asm__ __volatile__ ("fsave %0\n" : : "m"(fpubuf));
    	__asm__ __volatile__ ("frstor %0\n" : : "m"(fpubuf));
        }
```

in a signal handler, the system (or, at least, the CPU that was running the code) will freeze up hard. Ways of locking up the system from an unprivileged user-space program are generally considered to be bad news; they also, in general, are not seen as compiler bugs. A bit of digging turned up the real problem, and the latest kernel denial of service vulnerability was found. 

In theory, the `fsave` instruction above saves the floating-point unit (FPU) status into the `fpubuf` array; the subsequent `frstor` should simply restore the same state back into the FPU. Unfortunately, the above code is incorrect; the assembly instructions should read `"m"(*fpubuf)` to actually store the state into the `fpubuf` array. The code, as written, restores from the wrong address, corrupting the state of the FPU and, in particular, setting some exception flags. 

FPU exceptions do not result in immediate kernel traps; instead, the trap happens when the next floating-point command is executed. As it happens, the kernel checks when a signal handler returns and, if that handler has used any floating-point instructions, the kernel performs an `fwait` instruction to ensure that the last operation is complete. That `fwait` causes the floating point exception caused by the corrupt restore to be delivered as a kernel trap. 

The kernel has a way of dealing with floating point traps; it saves the FPU state and queues up a floating point exception signal for the current process. It also sets the TS ("task switched") processor flag to indicate that the FPU state may be other than expected. At that point, it returns to the place where the exception occurred. 

Normally, as part of returning from the trap, the kernel would simply deliver the floating-point exception signal to user space and get on with life. But, in this case, the kernel is returning back to kernel space, and back to the same `fwait` instruction that caused the problem in the first place. That instruction sees the TS flag and generates another trap. The handler for this trap knows just what to do in response to a TS flag; it restores the saved FPU state and returns. The saved FPU state is, however, the corrupted state which was in effect before the first attempt to execute `fwait`. So, at this point, the loop is closed and a new floating-point trap will be generated. This will go on for a while. 

The [fix](<https://lwn.net/Articles/89595/>) is relatively straightforward, once the problem is understood. The kernel simply clears any pending exceptions before executing `fwait`, and the problem goes away. All that is left is the updating and rebooting of large numbers of vulnerable systems. 

(Thanks to Sergey Vlasov, whose [analysis](<https://lwn.net/Articles/89771/>) of the problem made this article much easier to write.)
