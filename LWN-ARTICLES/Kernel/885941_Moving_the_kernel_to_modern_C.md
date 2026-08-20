---
title: Moving the kernel to modern C
url: https://lwn.net/Articles/885941/
date: "February 24, 2022"
category: Build system; GCC
author: "By Jonathan Corbet February 24, 2022"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 24, 2022

Despite its generally fast-moving nature, the kernel project relies on a number of old tools. While critics like to focus on the community's extensive use of email, a possibly more significant anachronism is the use of the 1989 version of the C language standard for kernel code — a standard that was codified before the kernel project even began over 30 years ago. It is looking like that longstanding practice could be coming to an end as soon as the 5.18 kernel, which can be expected in May of this year. 

#### Linked-list concerns

The discussion started with [this patch series](<https://lwn.net/ml/linux-kernel/20220217184829.1991035-1-jakobkoschel@gmail.com/>) from Jakob Koschel, who is trying to prevent speculative-execution vulnerabilities tied to the kernel's linked-list primitives. The kernel makes extensive use of doubly-linked lists defined by [`struct list_head`](<https://elixir.bootlin.com/linux/v5.17-rc5/source/include/linux/types.h#L177>): 

```
struct list_head {
    	struct list_head *next, *prev;
        };
```

This structure is normally embedded into some other structure; in this way, linked lists can be made with any structure type of interest. Along with the type, the kernel provides [a vast array of functions and macros](<https://elixir.bootlin.com/linux/v5.17-rc5/source/include/linux/list.h>) that can be used to traverse and manipulate linked lists. One of those is [`list_for_each_entry()`](<https://elixir.bootlin.com/linux/v5.17-rc5/source/scripts/kconfig/list.h#L43>), which is a macro masquerading as a sort of control structure. To see how this macro is used, imagine that the kernel included a structure like this: 

```
struct foo {
        	int fooness;
    	struct list_head list;
        };
```

The `list` member can be used to create a doubly-linked list of `foo` structures; a separate `list_head` structure is usually declared as the beginning of such a list; assume we have one called `foo_list`. Traversing this list is possible with code like: 

```
struct foo *iterator;
    
        list_for_each_entry(iterator, &foo_list, list) {
        	do_something_with(iterator);
        }
        /* Should not use iterator here */
```

The `list` parameter tells the macro what the name of the `list_head` structure is within the `foo` structure. This loop will be executed once for each element in the list, with `iterator` pointing to that element. 

Koschel included [a patch](<https://lwn.net/ml/linux-kernel/20220217184829.1991035-4-jakobkoschel@gmail.com/>) fixing a bug in the USB subsystem where the iterator passed to this macro was used after the exit from the macro, which is a dangerous thing to do. Depending on what happens within the list, the contents of that iterator could be something surprising, even in the absence of speculative execution. Koschel fixed the problem by reworking the code in question to stop using the iterator after the loop. 

#### The plot twists

Linus Torvalds [didn't much like the patch](<https://lwn.net/ml/linux-kernel/CAHk-=wg1RdFQ6OGb_H4ZJoUwEr-gk11QXeQx63n91m0tvVUdZw@mail.gmail.com/>) and didn't see how it related to speculative-execution vulnerabilities. After Koschel [explained the situation further](<https://lwn.net/ml/linux-kernel/86C4CE7D-6D93-456B-AA82-F8ADEACA40B7@gmail.com/>), though, Torvalds [agreed](<https://lwn.net/ml/linux-kernel/CAHk-=wiyCH7xeHcmiFJ-YgXUy2Jaj7pnkdKpcovt8fYbVFW3TA@mail.gmail.com/>) that ""this is just a regular bug, plain and simple"" and said it should be fixed independently of the larger series. But then he wandered into the real source of the problem: that the iterator passed to the list-traversal macros must be declared in a scope outside of the loop itself: 

> The whole reason this kind of non-speculative bug can happen is that we historically didn't have C99-style "declare variables in loops". So list_for_each_entry() - and all the other ones - fundamentally always leaks the last HEAD entry out of the loop, simply because we couldn't declare the iterator variable in the loop itself. 

If it were possible to write a list-traversal macro that could declare its own iterator, then that iterator would not be visible outside of the loop and this kind of problem would not arise. But, since the kernel is stuck on the C89 standard, declaring variables within the loop is not possible. 

Torvalds said that perhaps the time had come to look to moving to the [C99 standard](<https://en.wikipedia.org/wiki/C99>) — it is still over 20 years old, but is at least recent enough to allow block-level variable declarations. As he noted, this move hasn't been done in the past ""because we had some odd problem with some ancient gcc versions that broke documented initializers"". But, in the meantime, the kernel has moved its minimum GCC requirement to version 5.1, so perhaps those bugs are no longer relevant. 

Arnd Bergmann, who tends to keep a close eye on cross-architecture compiler issues, [agreed](<https://lwn.net/ml/linux-kernel/CAK8P3a0DOC3s7x380XR_kN8UYQvkRqvE5LkHQfK2-KzwhcYqQQ@mail.gmail.com/>) that it should be possible for the kernel to move forward. Indeed, he suggested that it would be possible to go as far as the [C11 standard](<https://en.wikipedia.org/wiki/C11_\(C_standard_revision\)>) (from 2011) while the change was being made, though he wasn't sure that C11 would bring anything new that would be useful to the kernel. It might even be possible to move to [C17](<https://en.wikipedia.org/wiki/C17_\(C_standard_revision\)>) or even the yet-unfinished [C2x](<https://en.wikipedia.org/wiki/C2x>) version of the language. That, however, has a downside in that it ""would break gcc-5/6/7 support"", and the kernel still supports those versions currently. Raising the minimum GCC version to 8.x would likely be more of a jump than the user community would be willing to accept at this point. 

Moving to C11 would not require changing the minimum GCC version, though, and thus might be more readily doable. Torvalds [was in favor](<https://lwn.net/ml/linux-kernel/CAHk-=wicJ0VxEmnpb8=TJfkSDytFuf+dvQJj8kFWj0OF2FBZ9w@mail.gmail.com/>) of that idea: ""I really would love to finally move forward on this, considering that it's been brewing for many many years"". After Bergmann [confirmed](<https://lwn.net/ml/linux-kernel/CAK8P3a2b_RtXkhQ2pwqbZ1zz6QtjaWwD4em_MCF_wGXRwZirKA@mail.gmail.com/>) that it should be possible to do so, Torvalds [declared](<https://lwn.net/ml/linux-kernel/CAHk-=wh97QY9fEQUK6zMVQwaQ_JWDvR=R+TxQ_0OYrMHQ+egvQ@mail.gmail.com/>): ""Ok, somebody please remind me, and let's just try this early in the 5.18 merge window"". The 5.18 merge window is less than one month away, so this is a change that could happen in the near future. 

It is worth keeping in mind, though, that a lot of things can happen between the merge window and the 5.18 release. Moving to a new version of the language standard could reveal any number of surprises in obscure places in the kernel; it would not take many of those to cause the change to be reverted for now. But, if all goes well, the shift to C11 will happen in the next kernel release. Converting all of the users of `list_for_each_entry()` and variants (of which there are well over 15,000 in the kernel) to a new version that doesn't expose the internal iterator seems likely to take a little longer, though.
