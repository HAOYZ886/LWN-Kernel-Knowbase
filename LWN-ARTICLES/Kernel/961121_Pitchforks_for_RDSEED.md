---
title: Pitchforks for RDSEED
url: https://lwn.net/Articles/961121/
date: "February 8, 2024"
category: "Architectures-x86; Confidential computing; Random numbers"
author: "By Jonathan Corbet February 8, 2024"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 8, 2024

The generation of random (or, at least, unpredictable) numbers is key to many security technologies. For this reason, the provision of random data as a CPU feature has drawn a lot of attention over the years. A proper hardware-based random-number generator can address the problems that make randomness hard to obtain in some systems, but only if the manufacturer can be trusted to not have compromised that generator in some way. A recent discussion has brought to light a different problem, though: what happens if a hardware random-number generator can be simply driven into exhaustion? 

As background, it is worth looking at two related instructions provided on x86 systems: 

  * **`RDSEED`** is a true random-number generator; it reads entropy collected from the environment and stores a random bit pattern into a CPU register. It is provided primarily for the seeding of pseudo-random number generators or other applications where the highest-quality randomness is needed. 
  * **`RDRAND`** obtains a random pattern from a deterministic random-number generator built into the hardware and stores it into a register. The generator used by `RDRAND` is periodically reseeded from the source used by `RDSEED`. 

`RDRAND` is recommended for most uses; it is faster and can significantly stretch the amount of entropy that is available to the system. Within the kernel, `RDSEED` is used to seed the kernel's random-number generator, but for little else. An important point to note is that neither of these instructions is privileged; they are equally available to user-space code. Also important is the fact that either instruction can fail if there is not sufficient entropy available. (See [this page](<https://www.intel.com/content/www/us/en/developer/articles/guide/intel-digital-random-number-generator-drng-software-implementation-guide.html>) for lots of details on how these instructions work). 

On the x86 architecture, the kernel has a pair of functions, [`rdseed_long()` and `rdrand_long()`](<https://elixir.bootlin.com/linux/v6.7.4/source/arch/x86/include/asm/archrandom.h#L18>), that make use of the above instructions. Kirill Shutemov recently observed that, while `rdrand_long()` will retry ten times if `RDRAND` fails, `rdseed_long()` gives up immediately. He posted [a patch](<https://lwn.net/ml/linux-kernel/20240130083007.1876787-1-kirill.shutemov@linux.intel.com/>) adding a retry loop to `rdseed_long()`, and [another](<https://lwn.net/ml/linux-kernel/20240130083007.1876787-2-kirill.shutemov@linux.intel.com/>) to issue a warning if even ten tries were not enough to get a successful result. That is where the discussion began. 

#### Can they fail (and do we care)?

Initially, there was some uncertainty over whether those instructions can fail at all. `RDSEED` is able to produce entropy at a high rate, and `RDRAND` multiplies that entropy considerably, so exhausting them seems unlikely. Jason Donenfeld, who maintains the kernel's random-number generator, [worried](<https://lwn.net/ml/linux-kernel/CAHmME9qsfOdOEHHw_MOBmt6YAtncbbqP9LPK2dRjuOp1CrHzRA@mail.gmail.com/>) that, if the CPU's random-number generator could be driven to failure, that denial-of-service problems could result. Dave Hansen, though, thought that such worries were misplaced; he [spoke authoritatively](<https://lwn.net/ml/linux-kernel/88a72370-e300-4bbc-8077-acd1cc831fe7@intel.com/>): 

> Despite the SDM allowing it, we (software) need RDRAND/RDSEED failures to be exceedingly rare by design. If they're not, we're going to get our trusty torches and pitchforks and go after the folks who built the broken hardware. 
> 
> Repeat after me: 
>
>> Regular RDRAND/RDSEED failures only occur on broken hardware 

It seems, though, that there may just be a use for those pitchforks. Donenfeld [wrote a little program](<https://lwn.net/ml/linux-kernel/Zbjw5hRHr_E6k18r@zx2c4.com/>) to stress the hardware by repeatedly executing `RDRAND` and `RDSEED` instructions; he observed that `RDSEED` failed nearly 30% of the time — with a single-threaded test. The hardware random-number generator, though, is shared between all CPUs in a socket, so a multi-threaded test can be expected to show worse results. Indeed, Daniel P. Berrangé [reported](<https://lwn.net/ml/linux-kernel/Zbk6h0ogqeInLa_1@redhat.com/>) that, with a multi-threaded test, `RDSEED` only succeeded 3% of the time. Nobody was able to demonstrate `RDRAND` failures but, as Berrangé [pointed out](<https://lwn.net/ml/linux-kernel/Zbt7mXg9p6IOdcqp@redhat.com/>) later in the discussion, that situation could change with a larger number of threads. Elena Reshetova also [acknowledged](<https://lwn.net/ml/linux-kernel/DM8PR11MB575052B985CA97B29A443F9AE77C2@DM8PR11MB5750.namprd11.prod.outlook.com/>) that it may be possible to force `RDRAND` to fail. 

The clear outcome is that, with a determined effort, `RDSEED` can be made to fail and the reliability of `RDRAND` is not guaranteed. The logical next question is: how much of a concern is this? For most use cases, there is little to worry about. The kernel's random-number generator will continue to produce unpredictable output if the hardware random-number generator fails occasionally, even in the absence of other entropy sources. As Ted Ts'o [pointed out](<https://lwn.net/ml/linux-kernel/20240201044735.GC2356784@mit.edu/>), the kernel makes use of any other entropy sources it can find (such as the variation in interrupt timings) and is intended to be robust even if `RDRAND` were to turn out to be entirely compromised. Since most applications get their random data from the kernel, even an unreliable `RDRAND` should not be a concern. 

There is, however, one noteworthy exception: the use case known as [confidential computing](<https://en.wikipedia.org/wiki/Confidential_computing>) (sometimes referred to as "CoCo"), which is intended to guarantee the security and confidentiality of virtual machines even if they are running on a compromised or hostile host computer. Techniques like secure enclaves and encrypted memory have been developed to protect virtual machines from a prying host; these technologies may, someday, work as intended, but they are absolutely dependent on the availability of trustworthy random data. If a "confidential" virtual machine can be launched with a known random seed, the game may be over before it starts. 

Availability of entropy at boot time has long been a problem for Linux systems, so a number of mechanisms have been developed to seed the random-number generator as quickly as possible. These can include using random data injected by the bootloader and collecting entropy from the environment. A confidential-computing system, though, cannot trust inputs like that. The bootloader is under the host's control, but so is environmental entropy. As Reshetova [explained](<https://lwn.net/ml/linux-kernel/DM8PR11MB5750B861F7A105886AA5FCE4E77C2@DM8PR11MB5750.namprd11.prod.outlook.com/>), the host is able to control events like the expiration of timers and the delivery of interrupts. The only source of entropy that is not, at least theoretically, under the host's control is the hardware random-number generator. If that, too, is compromised, the entire confidential-computing dream falls apart. 

#### What to do

That dream has been somewhat controversial in kernel circles from the beginning, and this revelation has not helped; at one point Donenfeld [asked](<https://lwn.net/ml/linux-kernel/CAHmME9qMO7=RDR60bKJvpDTRokcKed_i0+7BbFD53_7o2OJ6-g@mail.gmail.com/>) directly: ""Is this CoCo VM stuff even real? Is protecting guests from hosts actually possible in the end?"" Most of the ensuing discussion, though, was focused on what the appropriate response should be. 

Outside of the confidential-computing use case, the consensus seems to be that little needs to be done. Adding a warning if `RDSEED` or `RDRAND` fail (as Shutemov proposed at the beginning of the discussion) is being considered, but even that is not clearly the right thing to do. Many systems run with [`panic_on_warn`](<https://docs.kernel.org/admin-guide/sysctl/kernel.html#panic-on-warn>) enabled; on such systems, a warning will cause a system crash. That would turn a random-number-generator failure into a denial-of-service problem. Even in this case, though, a failure during early boot seems worth a warning; if that happens, either the random-number-generator is simply broken, or there is clearly some sort of attack underway. 

When the system is running in a confidential-computing mode, though, the situation is a bit more complicated. Among other things, that is not a mode that the kernel as a whole recognizes; Ts'o [suggested](<https://lwn.net/ml/linux-kernel/20240202153927.GA119530@mit.edu/>) adding a global flag for this purpose, since other parts of the kernel are eventually likely to need it as well. But, even with that in place, there are a number of alternatives to consider; Donenfeld [spelled them out in detail](<https://lwn.net/ml/linux-kernel/CAHmME9oqM2a766dBK22-yKr8=2-icg=UkQzmBOF8G5Zh_Y9E9w@mail.gmail.com/>). They come down to whether the kernel should warn (or panic) on failure, whether that should happen at any time or only during early boot, and whether this response should change in "confidential-computing mode". 

The emerging consensus would seem to be that the first step is simply retrying a failing operation, as is already done for `RDRAND`. Even if a single `RDRAND` can be made to fail, doing so ten times in a row is a more difficult prospect. That said, one should remember that, as Reshetova [pointed out](<https://lwn.net/ml/linux-kernel/DM8PR11MB5750FDC379F54F8591246DF0E7432@DM8PR11MB5750.namprd11.prod.outlook.com/>), the host controls a guest's scheduling and could, in theory, interfere with a retry loop as well. If retries are forced to happen when the random-number generator is exhausted, they will never succeed. 

At a minimum, a warning should be issued if these instructions fail during early boot, since that is a clear sign that something is going wrong. For the confidential-computing case, a randomness failure means that the system is unable to provide the guarantees upon which the whole edifice is built, so the system should simply refuse to run. That could be achieved with a panic, or by simply looping on `RDRAND` indefinitely, locking up the virtual machine if the instruction never succeeds. 

Beyond that, there is not a whole lot that needs to be done. 

There is one part of the discussion that is not visible, though: this concern seems to have created a scramble within the CPU vendors to characterize the extent of the problem and figure out what, if anything, needs to be done about it. Lest one think this is an Intel-specific issue, Berrangé [reported](<https://lwn.net/ml/linux-kernel/ZcHoKUElwXGPzrWb@redhat.com/>) the ability to force high failure rates on some AMD processors as well. The outcome of those discussions may be some combination of documentation, microcode updates, and design changes in future processors. 

Meanwhile, the prospect of random-number-generator exhaustion need not be a big worry for most users; it seems unlikely that it can be used to threaten real-world systems. For the confidential-computing crowd, it is just another one of what is sure to be an unending list of potential threats that need to be mitigated. Fixes will be put into place, and we can all put our pitchforks away and go back to watching the argument over whether confidential computing is achievable in the first place.
