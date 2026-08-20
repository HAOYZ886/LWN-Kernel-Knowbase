---
title: "Constant-time instructions and processor optimizations"
url: https://lwn.net/Articles/921511/
date: "February 3, 2023"
category: Architectures; Cryptography
author: "By Jonathan Corbet February 3, 2023"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 3, 2023

Of all the attacks on cryptographic code, timing attacks may be among the most insidious. An algorithm that appears to be coded correctly, perhaps even with a formal proof of its correctness, may be undermined by information leaked as the result of data-dependent timing differences. Both Arm and Intel have introduced modes that are intended to help defend against timing attacks, but the extent to which those modes should be used in the kernel is still under discussion. 

#### Timing attacks

Timing attacks work by observing how much time is required to carry out an operation; if that time varies according to the data being operated on, it can be used to reconstruct that data. As a simplistic example, imagine a password-checking function that simply compares a provided string against a password stored in a (presumably) secure location. A logical implementation would be to start at the beginning, comparing characters, and return a failure status as soon as an incorrect character is found. That algorithm could be naively coded as: 

```
nchars = max(strlen(attempt), strlen(password));
        for (i = 0; i < nchars; i++)
            if (attempt[i] != password[i])
    	    return false;
        return true;
```

The time this check takes is thus a function of the number of correct characters at the beginning of the string. An attacker could use this information to reconstruct the password, one character at a time. Real-world timing attacks that can, for example, extract cryptographic keys have been demonstrated many times. 

In response, security-oriented developers have learned to avoid data-dependent timing variations in their code. In the example above, for example, the entire password string would be compared regardless of where the first wrong character is found. All of this careful work can be undermined, though, if the CPU this code runs on has timing artifacts of its own. It will surely come as a shock to LWN readers to learn that, in fact, CPUs do exhibit such behavior. 

The CPU vendors are not unaware of this problem or its importance. But they are also not unaware of how beneficial some optimizations that introduce timing differences can be for certain benchmark results. The usual tension between security and performance objectives comes into play here, and the CPU vendors have taken the usual way out: make the users figure out which they want to sacrifice to gain the other. 

#### Constant-time processor modes

Both Arm and Intel have thus introduced modes that guarantee that some instructions, at least, will execute in constant time regardless of what the operands are. In Arm's case, the mode is called [Data Independent Timing (DIT)](<https://developer.arm.com/documentation/ddi0595/2021-06/AArch64-Registers/DIT--Data-Independent-Timing>); Intel calls its mode [Data Operand Independent Timing (DOIT)](<https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/best-practices/data-operand-independent-timing-isa-guidance.html>) mode, often called DOITM. Neither mode is enabled by default. 

Back in August 2022, Eric Biggers [asked](<https://lwn.net/ml/linux-kernel/YwgCrqutxmX0W72r@gmail.com/>) whether these modes should be enabled by kernel. As a result of that discussion, [a patch by Ard Biesheuvel](<https://git.kernel.org/linus/01ab991fc0ee>) was merged to enable DIT for the arm64 architecture — but only when running in the kernel. DIT remains off for user space by default, but enabling it is an unprivileged (and cheap) operation on that architecture, so user-space developers can enable it easily when they feel the need. This feature will be part of the 6.2 kernel release; it has not, as of this writing, been backported to any stable updates. 

The story for x86 is less clear. DOITM is controlled by a model-specific register (MSR) and cannot be changed by user space. The August discussion wound down despite [an attempt](<https://lwn.net/ml/linux-kernel/Y1du6kdk/6L0O56a@sol.localdomain/>) by Biggers to restart it in October. He [returned in January](<https://lwn.net/ml/linux-kernel/20230125012801.362496-1-ebiggers@kernel.org/>), asking once again whether DOITM should be enabled by default for x86 — and advocating that it should: 

> Cryptography algorithms require constant-time instructions to prevent side-channel attacks that recover cryptographic keys based on execution times. Therefore, without this CPU vulnerability mitigated, it's generally impossible to safely do cryptography on the latest Intel CPUs. 

Dave Hansen was [less enthusiastic](<https://lwn.net/ml/linux-kernel/14506678-918f-81e1-2c26-2b347ff50701@intel.com/>) about the idea, even though he concluded that it was ""generally the right thing to do"". He pointed to language in Intel's documentation stating that DOITM only adds value for code that was specifically written with timing differences in mind: 

> Translating from Intel-speak: Intel thinks that DOITM purely a way to make the CPU run slower if you haven't already written code specifically to mitigate timing side channels. All pain, no gain. 
> 
> The kernel as a whole is not written that way. 

DOITM, he said, is only going to be useful for a small amount of carefully written cryptographic code in user space, and will only be a performance loss for everything else. He also noted that Intel explicitly warns that the performance impact of DOITM may be ""significantly higher on future processors"". The "Ice Lake" generation of processors is the first where DOITM makes any difference at all; constant-time operations are evidently the norm on earlier generations. 

Biesheuvel [argued](<https://lwn.net/ml/linux-kernel/CAMj1kXHcUdJQDzF1VYe8uP_-Lb1W6JXxmfTEDYGuMZqSy=2BqA@mail.gmail.com/>) that the value of DOITM extends beyond user-space cryptographic libraries, and that it is even more relevant to the kernel: 

> But for privileged execution, this should really be the other way around: the scope for optimizations relying on data dependent timing is exceedingly narrow in the kernel, because any data it processes must be assumed to be confidential by default (wrt user space), and it will probably be rather tricky to identify CPU bound workloads in the kernel where data dependent optimizations are guaranteed to be safe and result in a significant speedup. 

Biggers [questioned Hansen's focus on performance](<https://lwn.net/ml/linux-kernel/Y9MAvhQYlOe4l2BM@gmail.com/>), saying that the kernel operations that benefit most from data-dependent optimizations are almost certainly the ones that most need protection, since they are the ones that will show the strongest timing differences. He concluded: 

> I think the real takeaway here is that the optimizations that Intel is apparently trying to introduce are a bad idea and not safe at all. To the extent that they exist at all, they should be an opt-in thing, not out-opt. The CPU gets that wrong, but Linux can flip that and do it right. 

Hansen [answered](<https://lwn.net/ml/linux-kernel/8b2771ce-9cfa-54cc-de6b-e80ce7af0a93@intel.com/>) that the community has ""looked at how bad the cure is compared to the disease for *every* one of these issues"", referring to other types of hardware vulnerabilities. DOITM is different, he said, because it looks like the cure is likely to get worse over time rather than better; that makes it hard to come up with a reasonable policy in the kernel. He later [added](<https://lwn.net/ml/linux-kernel/16e3217b-1561-51ea-7514-014e27240402@intel.com/>) that, after discussions within Intel, he feels the kernel community should not jump to enable DOITM: ""Suffice to say that DOITM was not designed to be turned on all the time. If software turns it on all the time, it won't accomplish what it was designed to do."" Doing that, he [said](<https://lwn.net/ml/linux-kernel/c5809098-9066-d90d-1bcc-108a11525cac@intel.com/>), would deprive systems of the ""fancy new optimizations"" that are coming in the future. 

No conclusions have been reached this time either — at least, not yet. It has not helped that, so far, nobody has posted any benchmarks showing what the performance impact of DOITM is. Assuming that cost is not huge, though, it would be surprising if DOITM does not end up being enabled by default in the kernel, at least, with the ability for user space to enable it on demand. "Insecure by default" is rarely a way to impress users, after all.
