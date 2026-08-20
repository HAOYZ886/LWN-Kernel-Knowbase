---
title: Really fixing getrandom()
url: https://lwn.net/Articles/802360/
date: "October 17, 2019"
category: Random numbers
author: "By Jonathan Corbet October 17, 2019"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 17, 2019

The final days of the 5.3 kernel development cycle included [an extensive discussion of the `getrandom()` API](<https://lwn.net/Articles/800509/>) and the reversion of an ext4 improvement that was indirectly causing boot hangs due to a lack of entropy. Blocking filesystem improvements because they are too effective is clearly not a good long-term development strategy for the kernel, so there was a consensus that some sort of better solution had to be found. What was lacking was an idea of what that solution should be. It is thus surprising that the problem appears to have been dealt with in 5.4 with little in the way of dissent or disagreement. 

The root of the problem in 5.3 was the blocking behavior of [`getrandom()`](<http://man7.org/linux/man-pages/man2/getrandom.2.html>), which will prevent a caller from proceeding until enough entropy has been collected to initialize the random-number generator. This behavior was subjected to a fair amount of criticism, and few felt the need to defend it. But changing `getrandom()` is not easy; any changes that might cause it to return predictable "random" numbers risks creating vulnerabilities in any number of security-sensitive applications. So, while various changes to the API were proposed, actually bringing about those changes looks like a multi-year project at best. 

There is another way to ensure that `getrandom()` doesn't block during the bootstrap process: collect enough entropy to ensure that the random-number generator is fully initialized by the time that somebody needs it. That can be done in a number of ways. If the CPU has a hardware random-number generator, that can be used; some people distrust this solution, but mixing entropy from the hardware with other sources is considered to be safe by most, even if the hardware generator is somehow suspect. But not all CPUs have hardware random-number generators, so this solution is not universal in any case. Other potential solutions, such as having the bootloader or early user space initialize the random pool, can work in some situations but are not universal either. 

Another possible solution, though, is jitter entropy: collecting entropy from the inherently nondeterministic nature of current hardware. The timing of even a simple sequence of instructions can vary considerably as the result of multiple layers of cache, speculative execution, and other tasks running on the system. Various studies into jitter entropy over the years suggest that it is real; it might not be truly nondeterministic, but it is unpredictable, and data generated using jitter entropy can pass the various randomness test suites. 

Shortly after the 5.3 release, Thomas Gleixner [suggested](<https://lwn.net/ml/linux-kernel/alpine.DEB.2.21.1909290010500.2636@nanos.tec.linutronix.de/>) using a simple jitter-entropy mechanism to initialize the entropy pool. Linus Torvalds [described](<https://lwn.net/ml/linux-kernel/CAHk-=wgjC01UaoV35PZvGPnrQ812SRGPoV7Xp63BBFxAsJjvrg@mail.gmail.com/>) this solution as ""not very reliable"", but he clearly thought that the core idea had some merit. He proposed a variant of his own that, after some brief discussion, was [committed into the mainline](<https://git.kernel.org/linus/50ee7529ec45>) at the end of the 5.4 merge window. 

Torvalds's patch adds a new function, [`try_to_generate_entropy()`](<https://elixir.bootlin.com/linux/v5.4-rc1/source/drivers/char/random.c#L1753>), which is called if somebody is requesting random data and the entropy pool is not yet fully initialized. It is simple enough to discuss in detail: 

```
static void try_to_generate_entropy(void)
        {
    	struct {
    	    unsigned long now;
    	    struct timer_list timer;
    	} stack;
```

This function starts by declaring a timer on the stack, which is a relatively rare occurrence in the kernel. Putting it onto the stack was a deliberate choice, though; if the timer function runs on a different CPU, it will cause cache contention (and unpredictable timing) for accesses to this function's stack space. 

```
stack.now = random_get_entropy();
    
    	/* Slow counter - or none. Don't even bother */
    	if (stack.now == random_get_entropy())
    	    return;
    	timer_setup_on_stack(&stack.timer, entropy_timer, 0);
```

On most architectures, `random_get_entropy()` just reads the timestamp counter (TSC) directly. The TSC increments for every clock cycle, so two subsequent calls should always return different values. Just how different they will be, though, is where the unpredictability comes in. The code above is simply verifying that the system on which the kernel is running does indeed have a high-frequency TSC; without that, this algorithm will not work. Most current hardware does have a TSC, fortunately. Assuming the TSC check passes, the timer is prepared for use. 

```
while (!crng_ready()) {
    	    if (!timer_pending(&stack.timer))
    		mod_timer(&stack.timer, jiffies+1);
    	    mix_pool_bytes(&input_pool, &stack.now, sizeof(stack.now));
    	    schedule();
    	    stack.now = random_get_entropy();
    	}
```

This loop is the core of the jitter-entropy algorithm. The timer declared above is armed to expire in the near future (the next "jiffy", which be anytime between 0ms and 10ms); that expiration will happen by way of an interrupt and may occur on a different CPU. The expiration of the timer adds complexity to the instruction stream (and, hopefully, more randomness as well). 

Each time through the loop, the TSC is queried and its value is mixed into the entropy pool. If the timer has expired, it is re-armed to run again. Then the scheduler is invoked, adding more complex code and an unpredictable amount of execution before that call returns. The loop itself runs until the entropy pool is deemed to be initialized. Given that a lot can happen even in one jiffy, this loop can be expected to run quite a few times and mix many TSC readings into the entropy pool. 

```
del_timer_sync(&stack.timer);
    	destroy_timer_on_stack(&stack.timer);
    	mix_pool_bytes(&input_pool, &stack.now, sizeof(stack.now));
        }
```

The post-loop cleanup gets rid of the timer and mixes in the last bit of entropy. 

The timer function itself looks like this: 

```
static void entropy_timer(struct timer_list *t)
        {
    	credit_entropy_bits(&input_pool, 1);
        }
```

In other words, every time the timer expires, the entropy pool is deemed to have gained one bit of entropy. It takes 128 bits of entropy for the pool to be considered ready, so the jitter loop may have to run for up to 128 jiffies — potentially just over one second on a system with a 100HZ tick frequency — if the system in question has no other sources of entropy. That could result in a perceivable pause during the boot process, but it is far better than blocking outright. 

When Torvalds decided to merge this code, he [suggested](<https://lwn.net/ml/linux-kernel/CAHk-=wi0vxLmwEBn2Xgu7hZ0U8z2kN4sgCax+57ZJMVo3huDaQ@mail.gmail.com/>) that it might not be the definitive solution to the problem: 

I'm not saying my patch is going to be the last word on the issue. I'm _personally_ ok with it and believe it's not crazy, and if it then makes serious people go "Eww" and send some improvements to it, then it has served its purpose. 

He was thus following the time-honored practice of submitting a patch in the hope that it would inspire somebody to create a better one. Thus far, though, there has been little in the way of commentary on this change — somewhat surprising, given the diversity of opinions expressed earlier in the discussion. Unless somebody comes along with a better idea or shows that the entropy produced by this algorithm is somehow predictable, this change seems likely to stand. Hopefully it achieves the goal of preventing `getrandom()` blocking while, at the same time, ensuring that random numbers from the kernel are always truly unpredictable.
