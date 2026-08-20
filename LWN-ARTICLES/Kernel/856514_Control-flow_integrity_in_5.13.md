---
title: "Control-flow integrity in 5.13"
url: https://lwn.net/Articles/856514/
date: "May 21, 2021"
category: "Releases-5.13; Security-Control-flow integrity"
author: "By Jonathan Corbet May 21, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 21, 2021

Among the many changes merged for the 5.13 kernel is support for [the LLVM control-flow integrity (CFI) mechanism](<https://clang.llvm.org/docs/ControlFlowIntegrity.html>). CFI defends against exploits by ensuring that indirect function calls have not been redirected by an attacker. Quite a bit of work was needed to make this feature work well for the kernel, but the result appears to be production-ready and able to defend Linux systems from a range of attacks. 

#### Protecting indirect function calls

The kernel depends heavily on indirect function calls — calls where the destination address is not known at compile time. Device drivers, filesystems, and other kernel subsystems interface with the generic, core code by providing functions to be called to carry out specific actions. When the time comes to, for example, open a file (which may be a special file corresponding to a device), the core kernel will make an indirect call to the appropriate `open()` function defined in the [`file_operations` structure](<https://elixir.bootlin.com/linux/v5.13-rc2/source/include/linux/fs.h#L2022>) for the file in question. Indirect function calls allow for a clean separation between generic and low-level code. 

This mechanism is flexible and performs well, but it also makes those indirect calls into an attractive target for attackers. If an indirect call can be redirected to an attacker-chosen location, there are few limits to the disorder that can result. Changes over the years have made it hard for attackers to inject their own code into the kernel, but if they can force execution to an arbitrary location, that matters little. Note that an exploit need not redirect a call to the beginning of another function; it can jump to any arbitrary point within the kernel image. There is no shortage of useful targets for a corrupted indirect function call. 

CFI attempts to block this sort of exploit by restricting indirect calls to locations that are plausible targets. In this case, "plausible" means that the call goes to the beginning of an actual function, and that the target function has the same prototype as the caller was expecting. That is not a perfect test; there may be functions with the same prototype that will perform some sort of useful action for an attacker. But the result is still a massive reduction in the set of available targets, which will often be enough. 

This check is often called "forward-edge CFI", since it protects calls to functions. The corresponding "backward-edge" protection ensures that return addresses on the stack have not been tampered with. The patches merged for 5.13 are focused on the forward-edge problem. 

#### LLVM CFI in Linux

Specifically, this CFI implementation works by examining the full kernel image at link time; for this reason, link-time optimization must also be enabled to use it. Whenever a location is found where the address of a function is taken, LLVM makes a note of the function and its prototype. It then injects a set of "jump tables" into the built kernel, one for each encountered function prototype. So, for example, the `open()` function mentioned above is defined as: 

```
int (*open) (struct inode *inode, struct file *file);
```

There are many functions in the kernel matching this prototype that have their address taken to stuff into a `file_operations` structure somewhere. LLVM will collect them all into a single jump table, which is essentially a list of the addresses of these functions. 

The next step is to change all of the places where that function's address is taken, and replace the address with the corresponding location in the jump table. So an assignment like: 

```
func_ptr = my_open_function;
```

will result in assigning an address within the jump table to `func_ptr`. 

Finally, whenever an indirect function call is made, control goes to a special function called `__cfi_check()`; this function receives, along with the target address, the address of the jump table matching the prototype of the called function. It will verify that the target address is, indeed, an address within the expected jump table, extract the real function address from the table, and jump to that address. If the target address is not within the jump table, instead, the default action is to assume that an attack is in progress and immediately panic the system. There is a permissive mode selectable at configuration time that simply logs the error instead. 

#### Kernel-specific quirks

That severe response may be justified, but it would be awfully annoying if there were situations where the kernel makes an indirect call to a function that doesn't exactly match the prototype of the pointer being used. So, naturally, the kernel did exactly that. In pre-5.13 kernels, [`list_sort()`](<https://elixir.bootlin.com/linux/v5.12.4/source/lib/list_sort.c#L104>) was declared as: 

```
void list_sort(void *priv, struct list_head *head,
    		   int (*cmp)(void *priv, struct list_head *a, struct list_head *b))
```

The comparison function `cmp()` is passed in by the caller and is invoked, via an indirect call, to compare items in the list. Inside `list_sort()`, though, one sees this line: 

```
a = merge(priv, (cmp_func)cmp, b, a);
```

The `cmp_func()` type to which the function pointer is cast looks _almost_ like the prototype of `cmp()`, except that the two `list_head` pointers have the `const` attribute. That is enough to change the prototype of the function and, at run time, to cause a CFI failure. The [fix](<https://git.kernel.org/linus/4f0f586bf0c8>) that was adopted was to propagate the `const` attribute to the callers of `list_sort()` so that the cast of the function pointer became unnecessary. That, however, required changing callers in 40 different files across the kernel source. 

Another interesting quirk comes from the fact that the jump tables are built at link time. That works for the monolithic kernel, but loadable modules are linked separately. CFI in loadable modules works, but each module gets its own jump tables. Remember that function pointers are replaced by pointers into the jump tables; since modules have different jump tables, they will get different pointers as well. In other words, the values of two pointers to the same function will differ if one of them is in a loadable module. 

For the most part, things will work anyway; calls to those two different pointers will end up in the same place. But consider this line in [`__queue_delayed_work()`](<https://elixir.bootlin.com/linux/v5.12.4/source/kernel/workqueue.c#L1626>): 

```
WARN_ON_ONCE(timer->function != delayed_work_timer_fn);
```

This test was [added to the 3.7 kernel](<https://git.kernel.org/linus/d8e794dfd51>) in 2012 as a way to ""detect delayed_work users which diddle with the internal timer""; nearly nine years later one assumes that they have all been found, but the test remains. But, if CFI is in use, then the address for `delayed_work_timer_fn()` as seen from a loadable module will not be the same as the address seen from the core kernel; that will cause the test to fail. There are a couple of places in the kernel with tests like this; they have been "fixed" by simply disabling the test when CFI is configured in. 

Various other things needed to be fixed as well, including making provisions for parts of the code that absolutely must have a direct pointer to a function rather than to the jump table. CFI in the kernel only works for the arm64 architecture in 5.13; support for x86 is in the works but is not yet ready to be enabled. There doesn't seem to be much in the way of data regarding the performance impact of this feature, but the [LLVM page describing CFI](<https://clang.llvm.org/docs/ControlFlowIntegrity.html#performance>) says that its cost is ""less than 1%"". 

CFI looks like a new feature that could have some scary, sharp edges. It is worth noting though that Kees Cook, when he sent [the pull request](<https://lwn.net/ml/linux-kernel/202104261141.5B699FE%40keescook/>) asking that the patches (which were written by Sami Tolvanen) be merged, said that CFI ""has happily lived in Android kernels for almost 3 years"". It is, in other words, already widely deployed in the real world and probably doesn't have many surprises left to offer — except, perhaps, for attackers, who will find that many of their exploits no longer work.
