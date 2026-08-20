---
title: A RDRAND followup
url: https://lwn.net/Articles/963281/
date: "February 26, 2024"
category: "Architectures-x86; Confidential computing; Random numbers"
author: "By Jonathan Corbet February 26, 2024"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 26, 2024

In a recent episode, "[Pitchforks for RDSEED](<https://lwn.net/Articles/961121/>)", we learned that there was some uncertainty around whether hardware-based random-number generators on x86 CPUs could fail. Since the consequences of failure in some situations (confidential-computing applications in particular) can be catastrophic, there was some concern about this prospect and what to do about it. Since then, the situation has come a bit more into focus, and there would appear to be an agreed-upon plan for changes to be made to the kernel. 

At the end of the last article, it was noted that there were ongoing discussions within the CPU vendors about whether the `RDRAND` and `RDSEED` instructions could fail to generate a properly random number. Proceeding without a thorough understanding of the hardware is, after all, a way to obtain the wrong kind of random outcomes. On February 14, Elena Reshetova [came back with a summary](<https://lwn.net/ml/linux-kernel/DM8PR11MB57503A2BB6F74618D64CC44AE74E2@DM8PR11MB5750.namprd11.prod.outlook.com/>) of Intel's position on the matter: 

> The RdRand in a non-defective device is designed to be faster than the bus, so when a core accesses the output from the DRNG, it will always get a random number. As a result, it is hard to envision a scenario where the RdRand, on a fully functional device, will underflow. 

In other words, `RDRAND` can be expected to always succeed. The lower-level `RDSEED` instruction, instead, consumes entropy more directly and, as participants in the discussion had already shown, can fail if stressed too hard. 

Confidential-computing applications, at the core of their security model, require random data that is not under the control of the host system (or anybody else). Only a random-number generator built into the CPU itself will fill the bill; anything else may be subject to external manipulation. There seems to be universal agreement, though, that the data from `RDRAND` is good enough even for confidential computing, and thus there is no need to use `RDSEED`. So these applications _should_ be safe on Intel systems — safe from a random-number compromise anyway. 

Jason Donenfeld, the maintainer of the kernel's random-number generator, responded with [a brief patch series](<https://lwn.net/ml/linux-kernel/20240214195744.8332-1-Jason@zx2c4.com/>) making a couple of changes. One is to immediately issue a warning if an attempt to use `RDRAND` fails, on the assumption that, as described above, any such failure is indicative of a hardware problem. That patch also eliminated the loop that would retry the operation ten times, since even one failure suggests that the hardware is broken and retrying is unlikely to fix it. There was just one little problem with that change, as was [pointed out](<https://lwn.net/ml/linux-kernel/696a5d98-b6a2-43aa-b259-fd85f68a5707@amd.com/>) by Tom Lendacky: the claim about the robustness of `RDRAND` applies to _current_ CPUs. Older processors might behave differently, and there may be cases where the retry loop is indicated. So that patch appears to have been dropped for now. 

The other patch, applied to the confidential-computing support code, attempts to seed the kernel's random-number generator with 256 bits of `RDRAND` output at boot time. Should that attempt fail, the result will be an immediate system panic. Any such failure would indicate that a confidential-computing virtual machine lacks a reliable source of randomness and, thus, will be unable to achieve the intended level of confidentiality. In such cases, it is better to just stop immediately rather than run in a potentially compromised mode. 

That latter patch has been [reposted](<https://lwn.net/ml/linux-kernel/20240222020616.2315199-1-Jason@zx2c4.com/>) with a few tweaks, and has obtained Reviewed-by tags from a number of relevant developers. So it looks likely to be the end of this particular story — until, as always seems to be the case, somebody finds something else to worry about in the random-number generator.
