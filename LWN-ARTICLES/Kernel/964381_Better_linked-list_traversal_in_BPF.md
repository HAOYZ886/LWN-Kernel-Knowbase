---
title: "Better linked-list traversal in BPF"
url: https://lwn.net/Articles/964381/
date: "March 8, 2024"
category: "BPF-Loops; Releases-6.9"
author: "By Jonathan Corbet March 8, 2024"
---

By **Jonathan Corbet**  
March 8, 2024

Before loading a BPF program, the kernel must verify that the program is safe to run; among other things, that verification includes ensuring that the program will terminate within a bounded time. That requirement has long made writing loops in BPF a challenging task. The situation has improved over the years for some types of loops, but others — including linked-list traversal — are still awkward in BPF programs. A new set of BPF primitives aims to make life easier for this use case through the installation of what can be seen as a sort of circuit breaker. 

Even relatively simple loops can be hard for the verifier to handle. To the human eye, a loop like this looks safe: 

```
for (i = 1; i < 10; i++)
        	do_something(i);
```

It can be hard, though, for the verifier (which is dealing with lower-level code for the BPF virtual machine) to know that nothing will reset the value of the iteration variable in a loop, though; without that assurance, it cannot verify that the loop will terminate as expected. Over the years, a number of helpers have been added to make this kind of iteration easier; they include the [`bpf_loop()` function](<https://lwn.net/Articles/877062/>) and [generic iterators](<https://lwn.net/Articles/926041/>). This sort of bounded iteration is now relatively easy to do in BPF programs. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

If one is iterating through a linked list, though, there is no loop variable that can bound the number of times the loop will run. There is no way for the verifier to know about the properties of a list that a program would like to traverse. If the list is circular, traversal could go forever. That prospect makes the verifier grumpy, forcing developers to engage in workarounds that make them even grumpier. When Alexei Starovoitov recently [proposed a solution](<https://lwn.net/ml/bpf/20240227055235.23463-2-alexei.starovoitov@gmail.com/>) to this problem, he provided an example of the code needed (in current kernels) to go through a list stored in a [BPF arena](<https://lwn.net/Articles/961941/>): 

```
for (struct bpf_iter_num ___it __attribute__((aligned(8),
                                                      cleanup(bpf_iter_num_destroy))),
    		* ___tmp = (bpf_iter_num_new(&___it, 0, (1000000)),
                        	pos = list_entry_safe((head)->first,
                                                  typeof(*(pos)), member),
    	                (void)bpf_iter_num_destroy,
    		     	(void *)0);
    	bpf_iter_num_next(&___it) && pos &&
                ({ ___tmp = (void *)pos->member.next; 1; });
            pos = list_entry_safe((void __arena *)___tmp, typeof(*(pos)), member))
```

Briefly, this construct creates a new generic iterator (the `bpf_iter_num_new()` call) set for a maximum of 1,000,000 iterations. The `bpf_iter_num_next()` call increments that iterator and forces an exit from the loop if it goes too high. The iterator is never expected to reach anything close to the maximum value; it exists only to reassure the verifier that something will force the loop to end at some point. 

One might fairly conclude that this code is not pleasant to write — and even less pleasant to try to understand. But, as Starovoitov put it: ""Unfortunately every 'for' in normal C code needs an equivalent monster macro"". He initially proposed a solution (a function called `bpf_can_loop()`), but the shape of that solution changed fairly quickly. 

As of [the v6 patch set](<https://lwn.net/ml/bpf/20240306031929.42666-1-alexei.starovoitov@gmail.com/>), the first step is to create a bit of infrastructure in the form of a new BPF instruction called `may_goto`. This instruction has some interesting semantics. If the kernel sees a `may_goto` instruction in a code block, it will automatically reserve space for an iteration count on the stack. Each execution of `may_goto` increments that count and compares it to a kernel-defined maximum; if that maximum is exceeded, a `goto` will be executed to a point just far enough ahead to insert another `goto`. 

This instruction is used to create a macro called `cond_break` that turns into BPF code like this: 

```
may_goto l_break;
       		 goto l_continue;
        l_break: 	 break;
        l_continue:  ;
```

In words: the macro normally uses `may_goto` to cause (by way of a bit of a `goto` dance) a `break` to be executed when the loop count is exceeded. This macro could, in turn, be used in this sort of loop: 

```
for (ptr = first_item; ptr; ptr = ptr->next)
        {
            do_something_with(ptr);
    	cond_break;
        }
```

The presence of `cond_break` (which uses `may_goto`) in the loop causes stack space to be set aside for an iteration count; the maximum is set to `BPF_MAX_LOOPS`, which is defined as `8*1024*1024` in current kernels. Each execution of `cond_break` checks the iteration count and forces an exit from the loop if the maximum is exceeded. 

Should that forced exit ever happen, chances are good that something is going wrong. Either some sort of out-of-control loop has been created, or the list to process is too long and the traversal will not be completed as expected. But, again, in real programs, exceeding the loop count is not expected to ever happen. It exists only as a sort of circuit breaker to reassure the verifier that the loop is safe to run. Or, as Starovoitov [put it](<https://lwn.net/ml/bpf/CAADnVQKKFxioLAqLPNq7mvt4GOHpC0j80-SUYzYQkpno3d+49Q@mail.gmail.com/>): 

> In other words "cond_break" is a contract between the verifier and the program. The verifier allows the program to loop assuming it's behaving well, but reserves the right to terminate it. So [a] bpf author can assume that cond_break is a nop if their program is well formed. 

The promise of the BPF verifier — that it would be able to guarantee that BPF programs cannot harm the kernel — was always going to be hard to achieve without imposing significant limitations on developers. Much of the work on BPF over the years has been aimed at lifting some of those limitations, which have only become more onerous as the complexity of BPF programs has increased. As awkward as the new features may seem, they are less so than what came before. 

Still, there is room for improvement. Starovoitov said that relying on loop counts was not the best approach, and that ""the actual limit of BPF_MAX_LOOPS is a random number""; he suggested that the kernel may eventually implement a watchdog timer to simply interrupt programs that run for too long. That might remove some of the awkwardness, but would have some interesting implications; BPF programs are not written with the idea that they could be interrupted at an arbitrary point. Addressing that could take a while; in the meantime, there is `cond_break`. There do not seem to be objections to the changes, and the patch set [has been merged](<https://lwn.net/ml/bpf/170975102930.9518.8336084287032577662.git-patchwork-notify@kernel.org/>) into the bpf-next repository, so `cond_break` seems likely to show up in the mainline during the 6.9 merge window.
