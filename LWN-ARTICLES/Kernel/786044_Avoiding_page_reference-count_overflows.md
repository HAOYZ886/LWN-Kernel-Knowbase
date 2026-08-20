---
title: "Avoiding page reference-count overflows"
url: https://lwn.net/Articles/786044/
date: "April 16, 2019"
category: "Memory management-struct page; Reference counting; Security-Vulnerabilities"
author: "By Jonathan Corbet April 16, 2019"
---

By **Jonathan Corbet**  
April 16, 2019

The [5.1-rc5 announcement](<https://lwn.net/Articles/786002/>) mentioned ""changes all over"" and highlighted a number of the areas that had been touched. One thing that was _not_ mentioned there was the addition of four patches fixing a security-related issue in the core memory-management subsystem. The vulnerability is sufficiently difficult to exploit that almost nobody should feel the need to rush out a kernel update, but it is still interesting to look at as a demonstration of how things can go wrong. 

One of the many things crammed into [`struct page`](<https://elixir.bootlin.com/linux/v5.0/source/include/linux/mm_types.h#L31>) is a field called `_refcount`; it is an `atomic_t` variable that counts the number of references to the page in question. As an `atomic_t`, it's a signed, 32-bit quantity. As long as this count is nonzero, references to the page exist and it cannot be reused for other purposes; once it drops to zero, the page can be freed. 

As with any counter in the kernel, `_refcount` should be manipulated with care lest it overflow. If one looks at the definition of [`get_page()`](<https://elixir.bootlin.com/linux/v5.0/source/include/linux/mm.h#L968>), one will see this check: 

```
VM_BUG_ON_PAGE(page_ref_count(page) <= 0, page);
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

If the reference count is incremented beyond the maximum positive value that can be held in a 32-bit signed variable (2,147,483,647), it will wrap around to the largest-magnitude negative value (-2,147,483,648). That, of course, wanders into undefined behavior, so the compiler is entitled to set it to `0xdeadbeef` and erase the disk drives instead if it so chooses. With reasonable compilers and a two's complement representation, though, one can expect the count to go negative when it overflows; that is what the above test is checking for. 

There are a couple of interesting things to note about this test. One is that it happens _before_ `_refcount` is incremented, so it will only trigger on the second overflow. That is not a problem, though, since the system will behave as expected — even with negative reference counts — as long as the count is not incremented all the way back to zero. The other thing to note is that, while `VM_BUG_ON_PAGE()` will crash the kernel when the `CONFIG_DEBUG_VM` configuration option is selected, it does precisely nothing in production builds. So, on most systems, an overflow of the page reference count will go undetected. That is potentially bad: if the reference count can be incremented to the point where it returns to zero, the page will be freed while a vast number of references remain, and a variety of use-after-free vulnerabilities will result. Good things do not generally ensue after an event like that. 

That said, memory-management developers have had good reason to be unworried about this eventuality. Overflowing the reference count to zero would require creating a full four-billion references to the page in question (the negative reference counts for the second two-billion references would look weird, but things will work as intended), and that is not an easy thing to do. As it turns out, according to [this merge commit](<https://git.kernel.org/linus/6b3a70773630>), Jann Horn has figured out how to do it, and it was indeed not easy: 

To have more than four billion references to a page requires a minimum of 32GB of kernel memory just for the pointers to the pages, much less any metadata to keep track of those pointers. Jann needed a total of 140GB of memory and a specially crafted filesystem that leaves all reads pending (in order to not ever free the page references and just keep adding more). 

Most of us are unlikely to give such resources to an attacker, even when asked nicely. But a bug is a bug, and the developers who were privy to the information about Horn's exploit decided to fix it. The result was four commits (from Linus Torvalds and Matthew Wilcox) making this particular hole even harder to exploit. 

The "specially crafted filesystem" mentioned above is necessary because it is not easy for an attacker running in user space to create large numbers of references to a given page. Simply creating a lot of mappings using `mmap()` or `fork()` would run into other limits long before the reference count overflowed. One way that does seem to work, though, is to create large numbers of direct-I/O requests, each of which acquires a reference while the operation is underway. Should the filesystem be extremely slow to finish those operations — even slower than VFAT — the references could eventually add up to the magic number. 

The first step toward preventing such exploits was to create a macro to check whether a given reference count is getting close to going negative: 

```
#define page_ref_zero_or_close_to_overflow(page) \
    	    ((unsigned int) page_ref_count(page) + 127u <= 127u)
```

The `VM_BUG_ON_PAGE()` check shown above was then changed to use this macro. On systems where `VM_BUG_ON_PAGE()` does anything at all, this new test should prevent reference counts from going negative, thus stopping things far short of incrementing all the way to zero. 

That said, if `CONFIG_DEBUG_VM` is set on a target system, an attacker could still use this overflow for denial-of-service attacks. To prevent that, Torvalds [created a new `try_get_page()` function](<https://git.kernel.org/linus/88b1a17dfc3e>) that will refuse to acquire a reference if the count is close to overflowing. Then, [`get_user_pages()` was changed](<https://git.kernel.org/linus/8fde12ca79af>) to use `try_get_page()` and to fail the entire operation if the needed references cannot be acquired. With those changes in place, direct-I/O requests that threaten to overflow a reference count will simply fail, and the attacker will be left looking for other good uses for 140GB of free memory. Another possible exploit, using pipes, was [closed off](<https://git.kernel.org/linus/15fab63e1e57>) in a similar fashion by Wilcox. 

While these changes were added late in the release cycle like an urgent fix, this clearly isn't a vulnerability that has a lot of people worried. In fact, it was first discussed in January on the closed kernel security list, but then the developers involved [simply forgot about it](<https://lwn.net/ml/linux-kernel/CAHk-=wj7jgMOVFW0tiU-X+zhg6+Rn7mEBTej+f26rV3zXezOSA@mail.gmail.com/>) for a while. That lapse has now been rectified, which is a good thing; one never knows when somebody might discover an easier way to force the page reference count to overflow. When that happens, the patches merged for 5.1-rc5 should be able to prevent a severe compromise of the system.
