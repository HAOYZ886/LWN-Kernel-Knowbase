---
title: Spectre revisits BPF
url: https://lwn.net/Articles/860597/
date: "June 24, 2021"
category: "BPF-Security; Security-Meltdown and Spectre"
author: "By Jonathan Corbet June 24, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 24, 2021

It has been well over three years now since [the Spectre hardware vulnerabilities were disclosed](<https://lwn.net/Articles/742702/>), but Spectre is truly a gift that keeps on giving. Writing correct and secure code is hard enough when the hardware behaves in predictable ways; the problem gets far worse when processors can do random and crazy things. For an illustration of the challenges involved, one need look no further than the BPF vulnerability described in [this advisory](<https://lwn.net/ml/oss-security/CAHMfzJkhZ01FG62sfMdXayK_NwD3g=5NcpGmg+-PVZLBpjJ9Fw@mail.gmail.com/>), which was fixed in the 5.13-rc7 release. 

Attacks on Spectre vulnerabilities generally rely on convincing the processor to execute, in a speculative mode, a sequence of operations that cannot happen in real execution. A classic example is an out-of-range array reference, even though the code performs a proper bounds check. The erroneous access will be backed out once the processor figures out that it mispredicted the result of the bounds check, but the speculative access will leave traces in the memory caches that can be used to exfiltrate data. 

The BPF virtual machine has always been an area of special concern when it comes to defending against speculative-execution attacks. Most such attacks rely on finding a fragment of kernel code that can be made to do surprising things when the CPU is executing speculatively; kernel developers duly have made a concerted effort to eliminate such fragments. But BPF exists to enable the loading of code from user space that runs within the kernel context; that allows attackers to craft their own code fragments and avoid the tedious task of combing through the kernel code. 

Much work has been done in the BPF community to frustrate those attackers. For example, array indexes are ANDed with a bitmask so that they cannot reach outside of the array even speculatively, regardless of what value they may contain. But it can be hard to anticipate every case where the processor may do something surprising. 

#### The vulnerability

Consider, for example, the following fragment of code, taken directly from [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/patch/?id=9183671af6dbf60a1219371d4ed73e23f43b49db>) by Daniel Borkmann fixing this vulnerability: 

```
// r0 = pointer to a map array entry
        // r6 = pointer to readable stack slot
        // r9 = scalar controlled by attacker
        1: r0 = *(u64 *)(r0) // cache miss
        2: if r0 != 0x0 goto line 4
        3: r6 = r9
        4: if r0 != 0x1 goto line 6
        5: r9 = *(u8 *)(r6)
        6: // leak r9
```

Incidentally, the changelog for this patch is an outstanding example of how to document a vulnerability and its fix; it's worth reading in full. 

In normal (non-speculative) execution, the above code has a potential problem. The register `r9` contains an attacker-supplied value; that value is assigned to `r6` in line 3, which is then used as a pointer in line 5\. That value could point anywhere in the kernel's address space; this is just the sort of unconstrained access that the BPF verifier was designed to prevent, so one might think that this code would never be accepted by the kernel in the first place. 

The verifier, though, works by exploring all of the possible paths that execution of a BPF program could take. In this case, there is no possible path that executes both lines 3 and 5\. The assignment of the attacker-supplied pointer only happens if `r0` contains zero, but that value will prevent the execution of line 5\. The verifier thus concludes that there is no path that can result in the indirection of a user-supplied pointer and allows the program to be loaded. 

But that verification runs in the real world; different rules apply in the speculative world. 

Line 1 in the above code fragment references memory that an attacker will have taken pains to ensure is not currently cached, forcing a cache miss. Rather than wait for memory to fetch the value, though, the processor will continue speculatively, making guesses about how any conditional statements involving `r0` will play out. And those guesses, as it turns out, could well be that neither `if` condition (in line 2 or 4) will evaluate true and, thus, neither jump will be taken. 

How can that be? Branch prediction doesn't work by guessing a value for `r0` and checking the result; it is, instead, based on what the recent history of that particular branch has been. That history is stored in the CPU's "pattern history table" (PHT). But the CPU cannot possibly track every branch instruction in a large program, so the PHT takes the form of a hash table. An attacker can locate code in such a way that its branches land in the same PHT entries as the branches in the crafted BPF program, then use that code to train the branch predictor to make the desired guesses. 

Once the attacker has loaded the code, cleared out the caches, and fooled the branch predictor into doing silly things, the battle is over; the CPU will speculatively reference the attacker-supplied address. Then it's just a matter of leaking the results in any of the usual ways. It is a bit of a tedious process — but computers are good at following such processes without complaining. 

It is worth noting that this is not a hypothetical attack. According to the advisory, multiple proofs-of-concept were sent to the `security@kernel.org` list when this problem was reported. Some of them do not require the step of training the branch predictor (one such is provided in the above-linked commit). These attacks can read out any memory in the kernel's address space; given that all of physical memory is contained therein, there are no real limits to what can be exfiltrated. Since unprivileged users can load a few types BPF programs, root access is not needed to carry out this attack. This is, in other words, a serious vulnerability. 

#### Closing the hole

The fix in this case is relatively straightforward. Rather than prune paths that the verifier "knows" will not be executed, the verifier will simulate them speculatively. So, for example, when checking the path where `r0` is zero, the unfixed verifier would simply conclude that the test in line 4 must be true and not consider the alternative. With the fix, the verifier will look at the false path (which includes line 5), conclude that an unknown pointer is being used, and prevent the loading of the program. 

This change has the potential to block the loading of correct programs that could be run before, though it is hard to imagine real-world, non-malicious code that would include this kind of pattern. It will, of course, slow the verification process to force it to examine paths that cannot occur in normal program execution, but that's the speculative world we live in. 

This fix was merged into the mainline and can be found in the 5.13-rc7 release. It has since found its way into the 5.12.13 and 5.10.46 stable updates, but not (yet) into any of the earlier stable releases. With this change, those kernels are protected against yet another Spectre vulnerability, but it would be foolhardy to assume that this is the last one.
