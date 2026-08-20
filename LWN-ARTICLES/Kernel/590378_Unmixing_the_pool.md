---
title: Unmixing the pool
url: https://lwn.net/Articles/590378/
date: "March 12, 2014"
category: Random numbers
author: "By Jake Edge March 12, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
March 12, 2014

One of the more useful outcomes from the Snowden revelations may well be the increased scrutiny of the security of our systems. While no one would (sensibly) claim that all of the problems have been found and fixed, there has been improvement in many different areas over the last year or so. One of the areas that has received some attention recently has been the kernel's random number generation. Two recent patches continue that trend, though it is hard to claim that they are the direct result of the exposure of NSA (and other secret agency) spying. 

Both patches update the state of the "entropy pool" that is used to generate random numbers for both `/dev/random` and `/dev/urandom`. That pool is associated with a (conservative) estimate of the amount of entropy (i.e. state unknowable by an attacker) stored within it. Anything that is believed to truly add entropy gets credited in that estimate, while other, possibly even attacker-controlled, input is simply mixed into the pool without entropy credit. The entropy estimate is used to block `/dev/random` readers when the amount of data requested is larger than the amount of entropy in the pool. 

#### Adding `RDSEED`

The first patch is from H. Peter Anvin and it simply [adds support for the `RDSEED` instruction](<https://lwn.net/Articles/589402/>) to the kernel. `RDSEED` is an instruction being added to Intel processors that returns ""fully conditioned entropy that is suitable for use as seeds to a PRNG [pseudo-random number generator]"". The patches use four bytes of `RDSEED` output to mix into the entropy pool at boot time (with no entropy credit). In addition, four bytes are generated using the instruction once per second and then mixed into the pool. It is also used to do an "emergency refill" with 64 bytes of `RDSEED` output if `/dev/random` is about to block due to a lack of entropy credit to fulfill a request. In both of the latter cases, four bits of credit are given for each byte of `RDSEED` output that gets mixed into the pool. 

Some may not be convinced that a black-box hardware random number generator (RNG) buried inside an Intel chip should be given that much (or any) entropy credit. It is a difficult question, as there is no _technical_ barrier to the instruction returning known-to-the-NSA sequences and there is no way for anyone (at least anyone outside of Intel) to know for sure. While that may seem paranoid, many formerly paranoid scenarios have moved into the "plausible" category over the last year. That concern has not been raised about the `RDSEED` patches, however. 

#### Mixing and unmixing

The other patch, from Kees Cook, would [add some output](<https://lwn.net/Articles/590398/>) from a newly instantiated hardware RNG into the entropy pool. When the RNG is registered (via `hwrng_register()`), sixteen bytes of its output would get mixed into the pool, but without any entropy credit. Jason Cooper was [concerned](<https://lwn.net/Articles/590417/>) that even mixing these bytes into the pool could lead to problems: ""By adding this patch, even without crediting entropy to the pool, a rogue hwrng now has significantly more influence over the initial state of the entropy pools."" 

But Cook didn't see it as any different than mixing in other random or system-specific data at initialization time: 

Sure, and I don't want to be the one weakening the entropy pool. However, I think this patch is no different from the ones that stuff a NIC MAC into the pool -- it's taking something from my system that is unique or random and pushing the entropy seed around. It seems silly that if I've loaded the hwrng-tpm module, my system entropy pool isn't bumped. 

In addition, former random subsystem maintainer Matt Mackall [brought up](<https://lwn.net/Articles/590420/>) an important aspect of the design of the mixing function. Because it can be reversed, mixing even attacker-controlled data into the pool can never remove randomness that was there at the outset: 

The pool mixing function is intentionally _reversible_. This is a crucial security property. 

That means, if I have an initial secret pool state X, and hostile attacker controlled data Y, then we can do: 

X' = mix(X, Y) 

and 

X = unmix(X', Y) 

We can see from this that the combination of (X' and Y) still contain the information that was originally in X. Since it's clearly not in Y.. it must all remain in X'. 

That didn't entirely mollify Cooper, who was still concerned that built-in hardware RNGs would have their output mixed in early in the boot sequence. He was worried that those bytes could pollute the pool, but Mackall [reiterated](<https://lwn.net/Articles/590422/>) his argument, putting it in starker terms: 

If the pool is in an attacker-knowable state at early boot, adding attacker-controlled data does not make the situation any worse. In fact, if the attacker has less-than-perfect control of the inputs, mixing more things in will make things exponentially harder for the attacker. 

Put another way: mixing can't ever [remove] unknownness from the pool, it can only add more. So the only reason you should ever choose not to mix something into the pool is performance. 

While the mixing function design with reversibility in mind (and its implications) was clear in Mackall's mind, it would seem that others in the kernel community did not know about it. It's an interesting property that makes perfect sense, once you know about it, but is rather counter-intuitive otherwise. In any case, Cooper's objections were withdrawn, and hardware RNG maintainer Herbert Xu [queued the patch](<https://lwn.net/Articles/590423/>). We should see it in 3.15.
