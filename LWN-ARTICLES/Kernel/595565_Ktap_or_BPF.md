---
title: "Ktap or BPF?"
url: https://lwn.net/Articles/595565/
date: "April 23, 2014"
category: "BPF-Tracing; Development tools-Kernel tracing; Tracing-with BPF"
author: "By Jonathan Corbet April 23, 2014"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 23, 2014

While the kernel's built-in tracing mechanisms have advanced considerably over the last few years, the kernel still lacks a DTrace-style dynamic tracing facility. In the last year we have seen the posting of two different approaches toward scriptable dynamic tracing: [ktap](<https://lwn.net/Articles/551314/>) and [BPF tracing filters](<https://lwn.net/Articles/575531/>). Both work by embedding a virtual machine in the kernel to execute scripts, but the similarity ends there. Putting one virtual machine into the kernel for tracing is a hard sell; adding two of them is not really seen as an option by anybody involved. So, at some point, a decision will have to be made. A recent discussion on that topic gives some hints about the direction that decision could go. 

The trigger for the discussion was the posting of [a new version of the ktap patch set](<https://lwn.net/Articles/592438/>) after a period of silence. While quite a bit of work has been done on ktap, little was done to address the concerns that [kept ktap out of the 3.13 kernel](<https://lwn.net/Articles/572788/>). Ingo Molnar, who blocked the merging of ktap the last time around, [was not pleased](<https://lwn.net/Articles/595580/>) that progress had not been made on that front. 

#### Virtual machines

There appear to be two specific points of argument that come up when the merits of ktap and BPF tracing filters are discussed. The first of those is, naturally, the question of introducing another virtual machine into the kernel. On this point, the discussion has shifted a bit, though, for a simple reason: while ktap needs its own virtual machine, the BPF engine is already in the mainline kernel, and it has been getting better. 

BPF originally stood for "Berkeley packet filter"; it was used as a way to tell the kernel how to narrow down a stream of packets from a network interface when tools like `tcpdump` are in use. Over time, though, BPF has been used in other contexts, such as filtering access to system calls as part of the [seccomp mechanism](<https://lwn.net/Articles/475043/>) and in a number of packet classification subsystems. Alexei Starovoitov's tracing filters patch set simply allows this virtual engine to be used to select and process system events as well. 

In 2011, BPF [gained a just-in-time compiler](<https://lwn.net/Articles/437981/>) that sped it up considerably. The 3.15 kernel takes this work further; it will feature a radically reworked (by Alexei) BPF engine that expands its functionality considerably. The new BPF offers the same virtual instruction set to user space, but those instructions are translated within the kernel into a format that is closer to what the hardware provides. The new format offers a number of advantages over the old, including ten registers instead of two, 64-bit registers, more efficient jump instructions, and a mechanism to allow kernel functions to be called from BPF programs. Needless to say, the additional capabilities have further reinforced BPF's position as the virtual machine of choice for an in-kernel dynamic tracing facility. 

Thus, if ktap is to be accepted into the kernel, it almost certainly needs to be retargeted to the BPF virtual machine. Ktap author Jovi Zhangwei [has expressed a willingness](<https://lwn.net/Articles/595581/>) to consider making such a change, but he sees a number of shortcomings in BPF that would need to be resolved first. BPF as it currently exists does not support features needed by ktap, including access to global variables, timer-limited looping (or loops in general, since BPF disallows them by design), and more. Jovi also repeatedly complained about the BPF tracing filter design, which is oriented around attaching scripts to specific tracepoints; Jovi wants a more flexible mechanism that would allow attaching a single script to a range of tracepoints. 

That last functionality should not be too hard to add. Most of the rest of Jovi's requests could probably be worked into BPF as well, especially if Jovi were to help to do the work. Alexei seems to be amenable to evolving BPF in ways that would enable it to better support ktap. The communication between the two developers appears to be difficult, though, with frequent misunderstandings being seen. At one point, Jovi [concluded](<https://lwn.net/Articles/595582/>) that Alexei was not interested in making the necessary changes to BPF; he responded by saying: 

Anyway, I think there will don't have any necessary to upstream ktap any more, I still enjoy the simplicity and flexibility given by ktap, and hope there will have a kernel built-in alternative solution in future. 

In truth, the situation need not be so grim, but there may be a need for an outside developer to come in and actually do the work to integrate ktap and BPF to show that it is possible. Thus far, volunteers to do this work have not made themselves known. And, in any case, there is another issue. 

#### Scripting languages

Ktap is built on the Lua language, which offers a number of features (associative arrays, for example) that can be useful in dynamic tracing settings. Ingo, along with a few others, would [rather see](<https://lwn.net/Articles/595583/>) a language that looks more like C: 

I'd suggest using C syntax instead initially, because that's what the kernel is using. The overwhelming majority of people probing the kernel are programmers, so there's no point in inventing new syntax, we should reuse existing syntax! 

The BPF tracing filters patch uses a restricted version of the C language; Alexei has also provided backends for both GCC and LLVM to translate that language into something the BPF virtual machine can run. So, once again, the BPF approach appears to have a bit of an advantage here at the moment. 

Unsurprisingly, Jovi [feels differently](<https://lwn.net/Articles/595584/>) about this issue; he sees the ktap language as being far simpler to work with. To support this claim, he provided this code from a BPF tracing filter example: 

```
void dropmon(struct bpf_context *ctx) {
            void *loc;
            uint64_t *drop_cnt;
    
            loc = (void *)ctx->arg2;
    
            drop_cnt = bpf_table_lookup(ctx, 0, &loc);
            if (drop_cnt) {
                __sync_fetch_and_add(drop_cnt, 1);
            } else {
                uint64_t init = 0;
                bpf_table_update(ctx, 0, &loc, &init);
            }
        }
```

This filter, he says, can be expressed this way in ktap: 

```
var s ={}
    
        trace skb:kfree_skb {
            s[arg2] += 1
        }
```

Alexei [concedes](<https://lwn.net/Articles/595585/>) that ktap has a far less verbose source language, though he has reservations about the conciseness of the underlying bytecode. In any case, though, he (along with others) has suggested that, once there is agreement on which virtual machine is to be used, there could be any number of scripting languages supported in user space. 

And that is roughly where the discussion wound down. There is a lot of interesting functionality to be found in ktap, but, the way things stand currently, it may well be that this code gets passed over in favor of an offering from a developer who is more willing to do what is needed to get the code upstream. That said, this discussion is far from resolved, and Jovi is not the only developer who is working on ktap. With the application of a bit of energy, it may yet be possible to get ktap's higher-level functionality into a condition where it could someday be merged.
