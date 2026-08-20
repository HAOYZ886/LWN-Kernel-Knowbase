---
title: sys_indirect()
url: https://lwn.net/Articles/259068/
date: "November 19, 2007"
category: Syslets; System calls; indirect
author: "By Jonathan Corbet November 19, 2007"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 19, 2007

Creating user-space APIs is a hard task. Even if an interface seems complete and well designed when it is created, the future can often add new requirements which the old API is hard-put to satisfy. So, for example, Unix started with the `wait()` system call. As applications got more complicated, it became necessary to wait for a specific process, to get more information about exiting processes, to wait in a non-blocking manner, and so on. So now, in addition to `wait()`, we have `waitid()`, `waitpid()`, and `wait4()`. Since old versions of system calls can (almost) never go away, changing needs over time tend to cause a proliferation of new calls. 

Most recently, Ulrich Drepper has been asking for the ability to add flags to system calls which create file descriptors, but which have no flags argument. Examples of these include `socket()` and `accept()`. It is possible to adjust the behavior of file descriptors created with these system calls after the fact (with `fcntl()`), but there will always be a period during which the file descriptors exist, but the desired behavior has not been set. When that behavior is "close on exec," and a multi-threaded program is running, one thread might run a new program with `exec()` before another one has managed to set the "close on exec flag." The result of this race is a leaked file descriptor which can, in turn, be a security problem. The only efficient way to close this particular race is for the kernel to create file descriptors with the desired flags set from the outset. 

Traditionally, this sort of problem would be solved through the creation of a new system call; one could, for example, add a four-argument `socket4()` which has the requisite `flags` parameter. This solution is unsatisfying, though; as has been seen, it leads to an ever-growing list of system calls. So it would be nice to find a different solution. Ulrich thinks he has done so by adding a single system call (`indirect()`), which works by passing additional information to existing system calls. 

It should be noted that [the first `sys_indirect()` implementation](<http://lwn.net/Articles/240329/>) was created by Davide Libenzi back in July. Ulrich wasn't entirely happy with that code, though: 

Davide's previous implementation is IMO far more complex than warranted. This code here is trivial, as you can see. I've discussed this approach with Linus last week and for a brief moment we actually agreed on something. 

The prototype for the new system call looks something like this: 

```
int indirect(struct indirect_registers *regs,
                     void *userparams,
    		 size_t paramslen,
    		 int flags);
```

The `regs` structure holds the process registers normally used in system calls; the system call number and its (normal) arguments, in other words. The extra parameters to be passed to the system call live in `userparams`, with a length of `paramslen`. The `flags` argument is currently unused; it's there for any sort of future expansion, since extending `indirect()` with itself is not allowed. 

The `task_struct` structure has been extended with a new field: 

```
union indirect_params indirect_params;
```

This union is meant to contain fields for each sort of parameter which can be added to a system call; in Ulrich's patch it looks like this: 

```
union indirect_params {
    	struct {
    	    int flags;
    	} file_flags;
        };
```

It can, thus, be used to pass a `flags` argument to system calls which deal in file descriptors. 

When `indirect()` is called, it checks the requested system call number against an internal whitelist. If the specific system call has not been marked as being extensible in this way, the call fails with `EINVAL`. Otherwise the application-supplied parameters are copied into the current process's `task_struct` structure and the system call is invoked in the usual way. Once that system call completes, the `indirect_params` area in the task structure is zeroed. 

The kernel provides no indication to the system call that it has been invoked via `indirect()`; the only difference in that case is that there might be non-zero values in `indirect_params`. So, in a sense, this mechanism can be seen as a way to add parameters to system calls with a default value of zero. So it is not possible, without some additional work, to add a parameter to a system call where passing a value of zero has a different meaning than omitting the parameter altogether. 

Should a need for yet another parameter materialize in the future, the size of the `indirect_params` structure can be increased as needed. As long as the kernel retains the old behavior when the new parameter has a value of zero, older applications and libraries will continue to operate as they did before. The extended system call need not (and cannot) know whether the larger `indirect_params` structure is being used or not. 

There is a possible use for this mechanism beyond extending system calls: the syslet developers see it as a possible way of specifying asynchronous behavior. The current syslet patches are essentially an indirect wrapper layer around system calls which specifies that the call is asynchronous (and what to do with the results). Adding two separate indirect layers for system calls seems like a suboptimal solution, so there is interest in adding syslet information to `indirect()` instead. That is one of the intended purposes for the currently-unused `flags` argument. 

Naturally, it would be surprising to see applications ever making calls to `indirect()`, well, directly. A much more likely scenario is for uses of `indirect()` to be buried inside the C library, which would then [export a more straightforward interface](<https://lwn.net/Articles/259345/>) to the application. 

While some developers (including Linus, evidently) like this patch set, others are less enthusiastic. David Miller [was blunt](<https://lwn.net/Articles/259346/>) in his review, saying: ""I think this indirect syscall stuff is the most ugly interface I've ever seen proposed for the kernel."" H. Peter Anvin is also [unimpressed](<https://lwn.net/Articles/259347/>): 

I think it is a horrible kluge. It's yet another multiplexer, which we are trying desperately to avoid in the kernel. Just to make things more painful, it is a multiplexer which creates yet another ad hoc calling convention, whereas we should strive to make the kernel calling convention as uniform as possible. 

So would it not be surprising if this new system call were to evolve somewhat before making its way into the mainline - it's a new and somewhat tricky API which could certainly benefit from discussion. But there are some real needs driving this work. So chances are that `indirect()` will eventually show up, in some form, in mainline kernels.
