---
title: Finding locking bugs with Smatch
url: https://lwn.net/Articles/1023646/
date: "June 11, 2025"
category: "Development tools-Static analysis"
author: "By Daroc Alden June 11, 2025 Linaro Connect"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Daroc Alden**  
June 11, 2025

* * *

[Linaro Connect](<https://lwn.net/Articles/1023503>)

[Smatch](<https://repo.or.cz/w/smatch.git>) is a GPL-licensed static-analysis tool for C that has a lot of specialized checks for the kernel. Smatch has [been used](<https://lwn.net/Articles/691882>) in the kernel for more than 20 years; Dan Carpenter, its primary author, decided last year that some details of its plugin system were due for a rewrite. He spoke at Linaro Connect 2025 about his work on Smatch, the changes to its implementation, and how those changes enabled him to easily add additional checks for locking bugs in the kernel. 

Video of the talk is [available](<https://www.youtube.com/watch?v=zy_u5pJn-UU>), and Carpenter's slides can be found [on Linaro's website](<https://resources.linaro.org/>). Carpenter began by apologizing for the relative complexity of this talk, compared to some of his presentations about Smatch in prior years. ""We're running out of easy checks to write,"" he explained. Smatch is designed to permit writing project-specific checks; over the years, a large number of kernel-specific checks have been added to the code, so the latest work has moved on to more complicated topics, such as locking. 

One of the things that sets Smatch apart from other static-analysis tools, Carpenter said, is its support for control-flow analysis and cross-function analysis. He frequently uses both of these features to understand new subsystems; Smatch can ""tell you where a variable is set, where callers of a function are, and what a function can return,"" among other things. For example, Smatch might show that a particular function has three callers, all of which hold a particular lock when they call it. From that, the programmer can infer the implicit locking requirements of the function. 

[ ![\[Dan Carpenter\]](https://static.lwn.net/images/2025/dan-carpenter-linaro-small.png) ](<https://lwn.net/Articles/1024165>)

That kind of report requires cross-function analysis, to trace nested calls with a lock held, but it also requires some starting knowledge of which functions acquire or release a lock. In the kernel, Smatch obtains that information from a hand-written table of every lock-related function, and then propagates that information through the different call trees. Rebuilding the complete database of calls takes multiple passes and five to six hours, which he said he does every night. 

Once the database is built, however, it allows files to be easily checked for common locking problems. The most common mistake is to fail to unlock a lock in a function's error path, he said. Smatch finds places where this has occurred by looking for functions that acquire a lock, and then have some possible control-flow path along which it is not released. That is slightly more complicated than it sounds. 

""It's harder than you might think to know what the start state is,"" he explained. If a function makes a call to `spin_lock()`, one might reasonably assume that the lock was not held before that point. But some functions behave differently depending on whether a lock is already held, so that is control-flow-sensitive as well. Also, sometimes a lock is referred to by multiple names, being locked by one name and unlocked via another. This complexity had resulted in Smatch's lock-tracking code slowly becoming an unreadable mess. ""And I'm the one who wrote it."" 

So, in the past year, Carpenter has rewritten everything. The reimplementation of the locking checks provides a blueprint for how to write modular Smatch checks, he said. Checks can now call `add_lock_hook()` and `add_unlock_hook()` to be informed when Smatch finds that a function call acquires or releases a lock somewhere in its call tree. Locks are also now tracked by type, instead of by name, in order to reduce problems with one lock being referred to by more than one name. 

There's a slight wrinkle with code that uses C's [ cleanup attribute](<https://gcc.gnu.org/onlinedocs/gcc/Common-Variable-Attributes.html#index-cleanup-variable-attribute>) to automatically unlock locks. On the one hand, it mostly eliminates bugs related to forgotten unlocks; on the other hand, it's a ""headache for static analysis"" since it makes the lock object harder to access and track. Ultimately, since they avoid many locking bugs, Smatch can ""mostly just ignore them"". 

Carpenter has used the new structure to write checks for double lock and unlock bugs as well. Unlike other static-analysis projects, Smatch focuses less on ""universal static properties"" and more on ""the actual bugs people are writing"". Smatch will not catch every possible double lock, double unlock, or forgotten unlock. That increases the number of false negatives from the tool, but it results in a much bigger reduction in false positives, he said. 

I asked how he found the classes of bugs that people actually write, in order to target them with Smatch checks. He explained that he reviews patches sent to the linux-stable mailing list in order to find bugs that could have been found earlier with static analysis. He encouraged other people to try the same thing, as he has found it educational. 

In the future, Carpenter wants to extend Smatch's double-lock checks to operate across function boundaries, to take advantage of Clang's upcoming support for tracking the relationship between locks and their data, and to handle lock-ordering bugs. 

As time wound down, one member of the audience wanted to know how Smatch compared to [ Cppcheck](<http://cppcheck.net/>), [ Coccinelle](<https://lwn.net/Articles/315686/>), and other static-analysis tools. Other open-source tools do not have good control-flow analysis, and practically no cross-function analysis, Carpenter said. Smatch does those things better, but it has its own weaknesses. The main problem is that it hasn't really been tested outside the kernel, he explained, so it's not clear how well Smatch will handle other styles of code. Smatch is also relatively slow. 

Coccinelle is fast, and can generate fixes for many problems. Sparse is good at finding endianness bugs and user-space pointers dereferenced in kernel space. ""But in terms of flow analysis, Smatch is the only tool that we have in open source."" 

After Carpenter's talk, I ran the tool against my own copy of the kernel; in that process, I learned that Smatch is best run from source. The last release predates Carpenter's rewrite, and there are a number of useful scripts included in the source distribution that are not present in the distribution packages. Smatch's [ terse documentation](<https://repo.or.cz/smatch.git/blob_plain/7a4fdad6e866cc5775322531813973e1f5c1d393:/Documentation/smatch.txt>) covers how to build its analysis database and run existing checks, but not how to query the database in a more free-form manner. The `smatch_data/db/smdb.py` script included in the source distribution can be used for that purpose. 

[Thanks to Linaro for sponsoring my travel to Linaro Connect.]
