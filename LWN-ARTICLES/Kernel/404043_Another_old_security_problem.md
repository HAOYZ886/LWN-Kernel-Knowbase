---
title: Another old security problem
url: https://lwn.net/Articles/404043/
date: "September 8, 2010"
category: "Security-Vulnerabilities"
author: "By Jonathan Corbet September 8, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 8, 2010

In August, a longstanding kernel security hole related to overflowing the stack area [was closed](<http://lwn.net/Articles/400746/>). But it turns out there are other problems in this area, at least one of which has been known about since late last year. Fixes are in the works, but it's hard not to wonder if we are not handling security issues as well as we should be. 

Once again, the problem was reported by Brad Spengler, who posted [a short program](<http://www.grsecurity.net/~spender/64bit_dos.c>) demonstrating how easily things can be made to go wrong. The program allocates a single 128KB array, which is filled as a long C string. Then, an array of over 24,000 `char *` pointers is allocated, with each entry pointing to the large string. The final step is to call `execv()`, using this array as the arguments to the program to be run. In other words, the exploit is telling the kernel to run a program with as many huge arguments as it can. 

Once upon a time, the kernel had a limit on the maximum number of pages which could be used by a new program's arguments. This limit would have prevented any problems resulting from the sort of abuse shown by Brad's program, but it was [removed for 2.6.23](<http://thread.gmane.org/gmane.linux.ports.hppa/752>); it seems that any sort of limit made life difficult for Google. In its place, a new check was put in which looks like this (from `fs/exec.c`): 

```
/*
    	 * Limit to 1/4-th the stack size for the argv+env strings.
    	 * This ensures that:
    	 *  - the remaining binfmt code will not run out of stack space,
    	 *  - the program will have a reasonable amount of stack left
    	 *    to work from.
    	 */
    	rlim = current->signal->rlim;
    	if (size > ACCESS_ONCE(rlim[RLIMIT_STACK].rlim_cur) / 4) {
    		put_page(page);
    		return NULL;
    	}
```

The reasoning was clear: if the arguments cannot exceed one quarter of the allowed size for the process's stack, they cannot get completely out of control. It turns out that there's a fundamental flaw in that reasoning: the stack size may well not be subject to a limit at all. In that case, the value of the limit is `-1` (all ones, in other words), and the size check becomes meaningless. The end result is that, in some situations, there is no real limit on the amount of stack space which can be consumed by arguments to `exec()`. And, unfortunately, the consequences are not limited to the offending process. 

At a minimum, Brad's exploit is able to oops the system once the stack tries to expand too far. He [mentioned](<https://lwn.net/Articles/404053/>) the possibility of expanding the stack down to address zero - thus reopening the threat of null-pointer exploits - but has not been able to figure out a way to make such exploits work. The copying of all those arguments will, naturally, consume large amounts of system memory; due to another glitch, that memory use is not properly accounted for, so, if the out-of-memory killer is brought in to straighten things out, it will not target the process which is actually causing the problem. And, as if that were not enough, the counting and copying of the argument strings is not preemptible or killable; given that it can run for a very long time, it can be very hard on the performance of the rest of the system. 

Brad says that he first reported this problem in December, 2009, but got no response. More recently, he sent a note to Kees Cook, who posted [a partial fix](<https://lwn.net/Articles/404056/>) in response. That fix had some technical problems and was not applied, but Roland McGrath has posted [a new set of fixes](<https://lwn.net/Articles/404060/>) which gets closer. Roland has taken a minimal approach, not wanting to limit argument sizes more than absolutely necessary. So his patch just ensures that the stack will not grow below the minimum allowed user-space memory address (`mmap_min_addr`). That check, combined with the guard page added to the stack region by the August fix, should prevent the stack from growing into harmful areas. Roland has also added a preemption point to the argument-copying code to improve interactivity in the rest of the system, and a signal check allowing the process to be killed if necessary. He has not addressed the OOM killer issue, which will need to be fixed separately. 

Roland's patch seems likely to fix the worst problems, though some commenters feel that it does not go far enough. One assumes that fixes will be headed toward distribution kernels in the near future. But there are a couple of discouraging things to note from this episode: 

  * It seems that the code which is intended to block runaway resource use in a core Linux system call was never really tested at its extremes. The Linux kernel community does not have a whole lot of people who do this kind of auditing and testing, unfortunately; that leaves the task to the people who have an interest (either benign or malicious) in security issues. 

  * It took some nine months after the initial report before anybody tried to fix the problem. That is not the sort of rapid response that this community normally takes pride in. 

The problem may indicate a key shortcoming in how Linux kernel development is supported. There are thousands of developers who are funded to spend at least some of their time doing kernel work. Some of those are paid to work in security-related areas like SELinux or AppArmor. But it's not at all clear that anybody is funded simply to make sure that the core kernel is secure. That may make it easier for security problems to slip into the kernel, and it may slow down the response when somebody points out problems in the code. There is a strong (and increasing) economic interest in exploiting security issues in the kernel; perhaps we need to find a way to increase the level of interest in preventing these issues in the first place.
