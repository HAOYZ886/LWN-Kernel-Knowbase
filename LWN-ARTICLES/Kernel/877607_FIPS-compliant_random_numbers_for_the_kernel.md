---
title: "FIPS-compliant random numbers for the kernel"
url: https://lwn.net/Articles/877607/
date: "December 7, 2021"
category: Development model; Random numbers
author: "By Jake Edge December 7, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
December 7, 2021

The Linux random-number generator (RNG) seems to attract an [outsized amount of attention](<https://lwn.net/Kernel/Index/#Random_numbers>) (and work) for what is, or seemingly should be, a fairly small component of the kernel. In part that is because random numbers, and their quality, are extremely important to a number of security protections, from unpredictable IP-packet sequence numbers to cryptographic keys. A recent [post](<https://lwn.net/ml/linux-kernel/2036923.9o76ZdvQCi@positron.chronox.de/>) of version 43 of the Linux Random Number Generator (LRNG) by Stephan Müller is not likely to go any further than its predecessors, but the discussion around it may lead to support for a feature that some distributions need. 

The cover letter for the LRNG patch set is titled ""/dev/random - a new approach"", which is true, but also sure to elicit highly skeptical responses or cause the patches to be ignored entirely. As was reiterated in the discussion, kernel development generally does not proceed along the "wholesale replacement" path; features are added slowly, in bite-sized chunks, instead. But LRNG is meant to be a drop-in replacement for the existing kernel RNG, while adding a long list of additional features—some of which would likely be welcome if they were separated out. 

Müller pointed to a [set of presentation slides](<https://www.chronox.de/lrng/doc/lrng_presentation_v43.pdf>) for a good overview of LRNG. One area where the kernel RNG has had difficulties over the years is in gathering enough entropy to provide cryptographic-strength random numbers at boot time, especially for virtual machines and systems without much entropy from disk interrupts (e.g. using SSDs). LRNG collects entropy faster at boot time using [CPU execution-time jitter](<https://lwn.net/Articles/642166/>) and other techniques. As described in his [LRNG paper](<https://www.chronox.de/lrng/doc/lrng.pdf>), this entropy collection complies with the [SP 800-90B](<https://csrc.nist.gov/publications/detail/sp/800-90b/final>) standard from the US National Institute of Standards and Technology (NIST). In addition, LRNG uses techniques to combine entropy sources in a fully documented, mathematical approach, rather than the informal mechanism in the current kernel RNG. 

The [first patch](<https://lwn.net/ml/linux-kernel/4641592.OV4Wx5bFTl@positron.chronox.de/>) covers the gathering and handling of entropy by the LRNG framework. It provides a deterministic RNG (DRNG) that is compliant with [NIST SP 800-90A](<https://csrc.nist.gov/publications/detail/sp/800-90a/rev-1/final>), but it allows for other DRNG implementations to be used within the framework. But the fact that it is meant as a drop-in replacement for the existing RNG means that it _replaces_ all of that code. The existing code works, with some known limitations, perhaps, but "starting over" with a new implementation has its own set of dangers. 

Beyond that, several in the discussion were skeptical about the value of the NIST standards (also known as [FIPS standards](<https://csrc.nist.gov/CSRC/media/Publications/fips/140/2/final/documents/fips1402annexc.pdf>)); Jason A. Donenfeld [said](<https://lwn.net/ml/linux-kernel/CAHmME9oaS4TOpk7rQ73BRKeVLjMUNyt6EFyeOX=hZSkFBPDu0g@mail.gmail.com/>): 

> You've posted it again, and yet I still believe this is not the correct design or direction. I do not think the explicit goal of extended configurability ("flexibility") or the explicit goal of being FIPS compatible represent good directions, and I think this introduces new problems rather than solving any existing ones. While there are ways the current RNG could or even should be improved -- or rewritten \-- this approach is still not that, no matter how many times you post it. 

But Müller [noted](<https://lwn.net/ml/linux-kernel/2560758.ogP2UNPRoF@tauon.chronox.de/>) that some distributions are carrying patches to comply with FIPS, which has led to fragmentation of the cryptographic RNG in the kernel. The idea behind LRNG is to try to ensure that whichever DRNG is chosen, the resulting random numbers are generated in a secure way. In addition, he said that he had not received complaints about the LRNG design, while he had incorporated lots of changes along the way as suggested by various Linux developers. The [changelog for the patch set](<https://www.chronox.de/lrng.html>), which goes back to 2016, shows quite a few changes of that sort. 

#### Distributions and FIPS

Greg Kroah-Hartman [wanted to know](<https://lwn.net/ml/linux-kernel/YZsyZua9T8DD6JF5@kroah.com/>) about distributions carrying patches for FIPS compliance and wondered: ""Why have the distros not submitted their changes upstream?"" Simo Sorce, who is on the RHEL crypto team, [answered](<https://lwn.net/ml/linux-kernel/afdf9c4a4005f6aeaded9e976c48160933f3c447.camel@redhat.com/>): ""We have not proposed them because they are hacks, we know they are hacks, and we know they are not the long term solution."" Red Hat does need some way to have FIPS compliance in its products, he said. 

But Kroah-Hartman [said](<https://lwn.net/ml/linux-kernel/YaEAbr%2Fvk621GpMj@kroah.com/>): ""Hacks that work today are the step toward a real solution."" He reiterated that evolution is what is needed to get FIPS compliance into the kernel, rather than completely replacing the random-number subsystem. ""Work off of those known-working-and-certified hacks. Submit them and let's go from there please."" 

Similarly, John Haxby [reported](<https://lwn.net/ml/linux-kernel/DBF8E8A8-9968-4E81-8C6E-7BAD0C3F39FA@oracle.com/>) that Oracle carries a [patch](<https://github.com/oracle/linux-uek/commit/5ebe83413c7573d566e581461bc95f9f139c5d4d>) to enable a FIPS-compliant RNG at boot time or by writing to a sysfs file; in FIPS mode, it always reseeds the DRNG from the jitter entropy. He said that it is ""not healthy"" for Oracle to carry out-of-tree patches like this, but it was expedient. He would rather have something upstream that is shared by everyone, but sees the patch as a temporary workaround: 

> We're carrying this patch simply because the certification requirements changed and this was the quickest and easiest way to workaround today's problem. It won't fix tomorrow's problem and next time we, and others, attempt FIPS certification then we, and others, will need a different /dev/random because neither the old one nor our quick and dirty workaround will actually be acceptable. 

Kroah-Hartman [suggested](<https://lwn.net/ml/linux-kernel/YaEACllCbkaHiXpX@kroah.com/>) that the patch was also a good starting point: ""Now that's a much smaller and simpler and easier to understand change, compared to 'rewrite the whole random number generator'."" He said that if those who need FIPS compliance worked together to get something working into the mainline, that would likely be an easier path. 

> But adding a stand-alone separate random subsystem just for this is not a good idea and is one huge reason why this patch set keeps being ignored by the kernel developers. 

Sorce also [replied](<https://lwn.net/ml/linux-kernel/56d2da397bb53f71c0354b102c3b40940e9b4eda.camel@redhat.com/>) to Donenfeld, noting that ""FIPS is essential for us and any design must include an option to be FIPS certifiable""; Müller has been working with distributions and standards organizations to gather and implement the requirements. In Sorce's other message, he described some of that work: 

> These patches have not been maturing in a void, but Stephan basically distilled discussions between multiple vendors as well as regulatory bodies (as you can see he has reviews from BSI and NIST requirements are also fully represented here). 
> 
> He addressed a few aspects I can mention but are not the only ones: performance (esp on NUMA systems), not blocking at boot due to lack of entropy, NIST/BSI conformance, flexibility so that future regulatory requirements can be easily integrated and upstreamed. 

#### More FIPS

Kroah-Hartman [would rather see](<https://lwn.net/ml/linux-kernel/YaEA0fyowmFlDfmK@kroah.com/>) the normal kernel development path followed here: ""Remember, evolution is the correct way of kernel development, not intelligent design :)"". But, as Müller [pointed out](<https://lwn.net/ml/linux-kernel/9311513.S0ZZtNTvxh@tauon.chronox.de/>), there was a [patch set](<https://lwn.net/ml/linux-kernel/20200921075857.4424-1-nstange@suse.de/>) posted for discussion in September 2020 to evolve the current RNG into one that was compliant, which never really went anywhere. Kroah-Hartman [asked](<https://lwn.net/ml/linux-kernel/YaT+9MueQIa5p8xr@kroah.com/>): ""That's a load of patches, some of them seem sane, what ever happened to them?"" The answer, Müller [said](<https://lwn.net/ml/linux-kernel/1852548.vyqkCxSGx7@tauon.chronox.de/>), is: ""Nothing was discussed, nothing was picked up."" 

Müller also said that LRNG does not actually replace the existing kernel RNG, it just provides a way for alternatives to be used: 

> One side note: the LRNG patch set does not replace random.c, but provides an additional implementation that can be selected at compile time. I am under the impression that is an equal approach considering other areas of the kernel like file systems, memory allocators, and similar. 

While it may make sense to have multiple upstream implementations in some areas, the kernel RNG is not one of those areas, Kroah-Hartman [said](<https://lwn.net/ml/linux-kernel/YaYvYdnSaAvS8MAk@kroah.com/>). Beyond that, the kernel RNG is used in multiple places in the kernel; ""Odds are, you REALLY do not want the in-kernel calls to be pulling from the 'random-government-crippled-specification' implementation, right?"" Sorce [did not agree](<https://lwn.net/ml/linux-kernel/ac123d96b31f4a51b167b4e85a205f31a6c97876.camel@redhat.com/>): 

> When our customers are mandated to use FIPS certified cryptography, they want to use it for kernel cryptography as well, and in general they want to use a certified randomness source as well. 

He understands the hesitancy to trust government agencies in light of problems like the [Dual EC DRBG mess](<https://lwn.net/Articles/578375/>), but the NIST specifications are not mandating a particular algorithm; the requirements are meant to allow multiple different implementations. Furthermore: 

> The specification is quite thorough and provides well reasoned requirements as well as self-test that insure coding mistakes won't end up returning non-random values. 

#### Maintainer questions

But there is another problem in following the evolution path to change the kernel RNG, Sorce said: 

> And the main question here is, how can we get there, in any case, if the maintainer of the random device doesn't even participate in discussions, does not pick obvious bug fixes and is simply not engaging at all? 
> 
> Your plan requires an active maintainer that guides these changes and interacts with the people proposing them to negotiate the best outcome. But that is not happening so that road seems blocked at the moment. 

Ted Ts'o is the maintainer of the kernel RNG, but has been notably absent in this and other discussions of changes and fixes for that subsystem. Kroah-Hartman [seemed skeptical](<https://lwn.net/ml/linux-kernel/YaZHKHjomEivul6U@kroah.com/>) that bug fixes were not being picked up, but Eric Biggers [listed](<https://lwn.net/ml/linux-kernel/YaZqVxI1C8RByq+w@gmail.com/>) several fixes and cleanups that had languished before eventually being picked up by other maintainers (one of them by Kroah-Hartman, in fact). Biggers concluded: ""So unfortunately, as far as I can tell, Ted is not maintaining random.c anymore."" 

Donenfeld [said](<https://lwn.net/ml/linux-kernel/CAHmME9p60Ve5XJTVcmGvSpUkg_hRp_i0rGG0R9VhuwLs0o_nXQ@mail.gmail.com/>) that he was willing to review fixes and improvements for the kernel RNG, but [cautioned](<https://lwn.net/ml/linux-kernel/CAHmME9qP9eYfPH+8eRvpx_tW8iAtDc-byVMvh4tFL_cABdsiOA@mail.gmail.com/>) that he is concerned that the FIPS requirements may be overbroad: 

> And so it would seem that the goal of implementing the RNG as best as we can might potentially be at odds with the goal of getting that green compliance checkbox, because that checkbox oversteps its bounds a bit. 
> 
> [...] I would like the kernel to have an excellent CSPRNG [cryptographically-secure pseudorandom number generator], from a cryptographic point of view, from a performance point of view, from an API point of view. I think these motivations are consistent with how the kernel is generally developed. And I think front loading the motivations with an external compliance goal greatly deviates and even detracts from the way the kernel is generally developed. 
> 
> [...] Specifically, I think that if you change your perspective from, "how can we change the algorithms of the RNG to be FIPS" to "how can we bend FIPS within its limits so that having what customers want would minimally impact the quality of the RNG implementation or introduce undue maintenance burdens." This means: not refactoring the RNG into some large abstraction layer that's pluggable and supports multiple different implementations, not rewriting the world in a massive patchset, not adding clutter. Instead, perhaps there's a very, very minimal set of things that can be done that would be considerably less controversial. 

Sorce was [amenable](<https://lwn.net/ml/linux-kernel/49d6091e571e24efff7bc4dc70c4c62628eb0782.camel@redhat.com/>) to that approach, and Haxby [said](<https://lwn.net/ml/linux-kernel/B4344F73-2591-40F0-AD5D-3E3C86D2288F@oracle.com/>) that he would submit the Oracle patch as a possible path forward. What we are seeing, at least in part, is a new maintainer [volunteering](<https://lwn.net/ml/linux-kernel/20211130184315.258150-1-Jason@zx2c4.com/>) to help out with the kernel RNG, which Ts'o [supports](<https://lwn.net/ml/linux-kernel/YauCkjsgDL4sdCId@mit.edu/>). If a more evolutionary approach, with reasoning beyond just "because FIPS", is proposed, it would seem that the kernel RNG may be able to check the compliance box without the upheaval that a full-on replacement could bring. Another possibility was [raised](<https://lwn.net/ml/linux-kernel/CACXcFmmzDrNQijjXmJaBwjXLiST_2LqVONpMO1JG5xvy-ZXhoQ@mail.gmail.com/>) by Sandy Harris; the FIPS requirements might be met with the existing RNG, but there are hurdles there as well: 

> [...] in fact their DRNG design requires an external source of random bits. However, it requires that the source be certified & that would be a problem for us. Intel & others might be able to get their random number instructions certified and vendors of crypto or SOC chips might get theirs certified, but the kernel community could not do that. 
> 
> I think the kernel's entropy collection routines are good enough that they could, in principle, be certified, but that would involve some work & considerable money. 

It seems clear that some solution is needed, at least for the enterprise distributions. Müller's patches provide a mechanism that is FIPS-compliant and apparently has minimal impact in terms of performance—perhaps even better performance than the existing implementation—while solving a number of other problems. The techniques used could form a basis for a relatively small number of changes that might benefit all users of the kernel RNG. The patch set as it stands now is not going to fly, but, with luck and some perseverance, the FIPS requirements could be met by following the usual kernel-development strategy. Only time will tell.
