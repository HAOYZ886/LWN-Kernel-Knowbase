---
title: Bounded loops in BPF programs
url: https://lwn.net/Articles/773605/
date: "December 3, 2018"
category: "BPF-Loops"
author: "By Jonathan Corbet December 3, 2018 Linux
Plumbers Conference"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 3, 2018

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2018-Linux_Plumbers_Conference>)

The BPF verifier is charged with ensuring that any given BPF program is safe for the kernel to load and run. Programs that fail to terminate are clearly unsafe, as they present an opportunity for denial-of-service attacks. In current kernels, the verifier uses a heavy-handed technique to block such programs: it disallows any program containing loops. This works, but at the cost of disallowing a wide range of useful programs; if the verifier could determine whether any given loop would terminate within a bounded time, this restriction could be lifted. John Fastabend presented a plan for doing so during the [BPF microconference](<http://vger.kernel.org/lpc-bpf.html>) at the 2018 [Linux Plumbers Conference](<https://linuxplumbersconf.org>). 

Fastabend started by noting that the lack of loops hurts; BPF developers are doing "crazy things" to work around their absence. He is working to enable the use of simple loops that can be modeled by the verifier. There is academic work on ways to verify more complex loops, but that is a problem for later. For now, the objective is to detect simple loops and verify that they will terminate; naturally, it's important that the verifier, too, is able to terminate in a reasonable amount of time. 

The key to determining the behavior of a loop is to find the induction variable that controls it. For a simple loop like: 

```
for (i = 0; i < MAX; i++)
    	/* loop body */
```

the induction variable is `i`. If the variable can be shown to be both monotonically increasing and bounded in value, then the verifier can conclude that the loop will terminate. Once the induction variable and its bounds have been identified, the verifier can also check that any memory references using that variable remain in range. 

This kind of verification could be done relatively easily with knowledge of where the loops are. But BPF is an unstructured virtual machine language that doesn't contain information about loops, so the verifier has to figure [![\[John Fastabend\]](https://static.lwn.net/images/conf/2018/lpc/JohnFastabend-sm.jpg)](<https://lwn.net/Articles/773612/>) out where they are itself. This is done by creating a [dominator tree](<https://en.wikipedia.org/wiki/Dominator_\(graph_theory\)>) that describes the program to be examined; it will identify tests that control the execution of specific blocks of code. From there, it is possible to identify loops (by looking for reverse jumps to the dominator node) and the blocks of code that belong to each loop. 

Doing so is not entirely easy. The use of a dominator tree requires that there be a single entry point into every loop; code that jumps into the middle of a loop cannot be verified. Identifying the looped-over code is also an expensive (O(n2)) algorithm. 

Fastabend outlined a few approaches to this problem, describing the first as the "by the book" method. This algorithm builds the dominator tree, then works to detect (and abort on) loops that cannot be reduced to a verifiable case. For each loop, it finds the induction variable, and verifies that variable's bounds; the execution of the loop is then simulated with the induction variable's largest and smallest values. The problem here is that the induction variable must be found with pattern matching, and the LLVM optimizer creates a wide variety of patterns that change with every release. That makes the code fragile. 

The next approach is to get the compiler to help by limiting the number of loop types that it generates. That makes the pattern-matching task easier, since the patterns that identify loops will be reduced in number and less prone to change. The verifier still has to do all of the work, but it becomes quite a bit more robust. 

But the best solution, he said, would be to create a new set of BPF instructions specifically to implement loops. They would mark the beginning and end of each loop, and include the test of the induction variable; the verifier would replace those instructions with actual tests and jumps. With these markers to denote the loop blocks, there would be no need for the dominator tree, since the markers themselves would make it clear which code is controlled by the loop. That would keep the verifier code minimal which, he said, is the right tradeoff in the end. 

The description of the session said that the goal ""to come to a consensus on how to proceed to make progress on supporting bounded loops"". That did not happen, but there was some discussion about the options and the development community is more aware of how this complex work is proceeding. In the end, real consensus is likely to come about in the usual way: through the posting of code that shows how the idea is implemented in the real world. 

The [slides from this talk [PDF]](<http://vger.kernel.org/lpc_bpf2018_talks/bounded-loops.pdf>) and [a video recording [YouTube]](<https://www.youtube.com/watch?v=PR4hpxWzPfo&t=4215
>) are available. Curious readers can also see [the implementation of the first alternative](<https://lwn.net/ml/netdev/20180601092646.15353.28269.stgit@john-Precision-Tower-5810/>) as it was posted in June. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to the event.]
