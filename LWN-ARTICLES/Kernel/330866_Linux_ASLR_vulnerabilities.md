---
title: Linux ASLR vulnerabilities
url: https://lwn.net/Articles/330866/
date: "April 29, 2009"
category: "Random numbers; Security-Vulnerabilities"
author: "By Jake Edge April 29, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
April 29, 2009

A recent LWN [comment thread](<https://lwn.net/Articles/328695/#Comments>)--which unfortunately descended into flames and rudeness--had a [post](<https://lwn.net/Articles/330488/>) with some interesting pointers to recent security research on Linux address space layout randomization (ASLR). Both look to be plausible attacks against ASLR, and have not yet been addressed by the kernel hackers. Perhaps worse than that, though, is that these kinds of problems are evidently not being reported to linux-kernel (or other kernel security channels), or not being acted on. Over the years, the interaction of security researchers and kernel hackers has often been contentious, to the point where some security researchers may not be reporting the Linux flaws they find via the usual channels. 

ASLR is a technique used to thwart buffer overflow vulnerabilities in user applications by randomizing the location of various pieces of the application's address space. Libraries, the heap and stack, as well as the executable code for a process are placed at random addresses so that attacker programs have a much more difficult time exploiting a buffer overflow. Without the use of ASLR, an attack could use hardcoded addresses of known locations in a process's address space (e.g. specific library functions) to perform its nefarious deeds. 

It is important that attacker programs be unable to see--or figure out--the memory layout for other processes in the system. Attackers who can gain that information could then use any buffer overflows they know of for that program with all of the addresses they need. For that reason, `/proc/pid/maps` (a file that describes the address space for process id `pid`) only contains data when read by the owner of that `pid`--or someone who can `ptrace()` it. A recent [advisory](<https://lwn.net/Articles/330877/>) about memcache and memcacheDB divulging that information, unauthenticated over the network should be worrisome for just this reason. 

The decision to stop allowing anyone to read the `maps` file came about in 2.6.22, long after ASLR was added in 2.6.12. Based on a [presentation [PDF]](<http://www.cr0.org/paper/to-jt-linux-alsr-leak.pdf>) at this year's CanSecWest conference, there is still enough information being leaked from `/proc` files to be able to determine the address space layout for a program. 

The `/proc/pid/stat` file reports the value of the instruction and stack pointers of the process, and the `/proc/pid/wchan` file reports its "wait channel", which is the function in which the process is currently blocked. Using that information, possibly sampled multiple times, along with a map of the instruction boundaries of the executable, Julien Tinnes and Tavis Ormandy were [able to bypass ASLR](<http://blog.cr0.org/2009/04/local-bypass-of-linux-aslr-through-proc.html>). 

The second flaw in ASLR was [presented at Black Hat Europe](<http://www.blackhat.com/html/bh-europe-09/bh-eu-09-speakers.html#Fritsch>) by Hagen Fritsch. A [whitepaper [PDF]](<http://www.blackhat.com/presentations/bh-europe-09/Fritsch/Blackhat-Europe-2009-Fritsch-Bypassing-aslr-whitepaper.pdf>) describing the flaw is instructive. Essentially, the random number generator (RNG) used to create the addresses for ASLR is flawed, allowing those values to be correctly calculated up to two minutes after a target process has been run. 

There is clearly a disconnect between the comment in the `get_random_int()` function (which uses the IP RNG `secure_ip_id()`) and the implementation of re-keying the RNG in `drivers/char/random.c`. The former claims that it gets re-keyed every second, but the `REKEY_INTERVAL` in the random driver is five _minutes_. If ASLR requires the RNG to re-key every second, a different function should be used. But, there is an additional problem. 

The `secure_ip_id()` function takes one argument which it mixes with the key in order to generate the random number. `get_random_int()` passes the sum of the `pid` and the internal kernel counter `jiffies` as that parameter. For a period of five minutes, if the attacker can arrange for the same sum to be passed in, they will get the same value as the target process did. That can happen in one of two ways: either by calling `execve()` on the desired target within one jiffy of when the attack process started--a rather difficult thing to arrange for a number of reasons--or by calling `execve()` when `pid + jiffies` is the same as it was for the target process. 

An attacker process can spawn children until it gets a desired `pid`, then wait for `jiffies` to reach a value where the sum is the same. Even though the absolute value of `jiffies` is not known outside of the kernel, various calculations on the difference in `jiffie` values can be used to narrow down the search. Once again, the `/proc/pid/stat` file can come into play here, by providing a start time for the target process with a granularity typically 2.5 times that of `jiffies` (10ms vs. 4ms). 

In addition, Fritsch notes that IP sequence numbers may be leaking information that could be used to assist in this attack because it uses the same RNG with the five minute re-key time. He has not looked at whether that is the case. 

These two vulnerabilities are fairly substantial and should certainly be fixed. It would seem fairly straightforward to limit access to the `/proc` files based on the same `ptrace()` test used for `maps`. The RNG flaw is more subtle and probably requires a fair amount of thought, but it is clear that the randomness provided is insufficient, at least for ASLR. 

Another [report](<http://kernelbof.blogspot.com/2009/04/kernel-memory-corruptions-are-not-just.html>) that came out of the comment thread demonstrates a misclassification of security flaws that tends to be very annoying to the security community. Misclassifying remotely exploitable flaws as a "denial of service" (due to a kernel crash) is a fairly common thing for distributions and others (knowingly or not) to do. As the blog posting indicates, it irritates some researchers: 

I'm wondering why kernel developers (or vendors?) continue to claim that kernel memory corruption are just Denial of Service. Most of the times they _are_ exploitable.. yes, even when the vulnerability is remotely triggered, yes.. even when the corruption takes place in a freaking slub in the middle of a kernel _heap_ .. yes even when you have kernel data pages marked NX and the kernel .text read-only and yes, absolutely yes even when you start only with a 16bit displacement... 

That particular vulnerability is long fixed in the kernel, but the whole posting is worth a read for those interested in how a kernel buffer overflow can become a remote root exploit (even bypassing SELinux). It is also indicative of the frustration that some in the security community feel about Linux security. For good or ill, Linux security is not well regarded in that community, to the point where it appears that some, possibly large, amount of Linux kernel security research is not being communicated to the kernel community. Perhaps that communication is occurring but is just "flying under the radar"--something that frequently happens with security discussions--as it would be a tragedy to think that known vulnerabilities are just falling through the cracks.
