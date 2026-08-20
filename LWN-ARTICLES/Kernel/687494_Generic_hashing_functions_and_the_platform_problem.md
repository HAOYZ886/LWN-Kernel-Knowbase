---
title: Generic hashing functions and the platform problem
url: https://lwn.net/Articles/687494/
date: "May 18, 2016"
category: Hashing functions
author: "May 18, 2016 This article was contributed by Neil Brown"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

May 18, 2016

This article was contributed by Neil Brown

What is a kernel developer to do when they need a simple hashing function to use in a new hash table, and they find that the obvious choice provided by the kernel works poorly? The "right" option is to fix the common code. The "easy" option is to write a replacement or a workaround. The "best" option, it seems, is to make sure Linus Torvalds finds out, because this is just the sort of thing that he cares about.

Linux has a fairly simple and efficient set of hashing functions in [`include/linux/hash.h`](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/include/linux/hash.h?id=b562e44f507e863c6792946e4e1b1449fbbac85d>) that work on simple input values: 32-bit or 64-bit numbers, or pointers that are cast to one of those depending on the architecture. The hash is computed by multiplying the input by one large number and ignoring any overflow, thus effectively taking the remainder of a division by another large number: 232 or 264, depending on the word size. The required number of bits are then extracted from the result.

When Thomas Gleixner was testing his [second patch set](<https://lwn.net/Articles/685425/>) for [making futexes faster](<https://lwn.net/Articles/685769/>), he discovered that the hash values returned weren't particularly evenly distributed and, as a result, he was receiving more collisions than expected. He addressed this problem by writing a simple alternative, sparking [a sub-thread](<http://www.gossamer-threads.com/lists/linux/kernel/2428809>) exploring the problems with these hash functions. As Torvalds found during subsequent investigations, Gleixner was not the first developer to decide it was easier not to fix the broken function. When working on the RAID 5/6 code for Btrfs, David Woodhouse, or possibly Chris Mason, discovered problems with the "`hash_64()`" function and helpfully [provided a comment](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/btrfs/raid56.c?id=53b381b3abeb86f12787a6c40fee9b2f71edc23b#n195>):

```
* we shift down quite a bit.  We're using byte
         * addressing, and most of the lower bits are zeros.
         * This tends to upset hash_64, and it consistently
         * returns just one or two different values.
```

It was determined that zeros in the lower-order bits result in the remaining bits not being mixed well, so there is not much variation in the output — not a particularly useful property for a hash function. The chosen response was not to fix `hash_64()` but to shift down the input to hide the problem.

Some eight years ago, Matthew Wilcox [made some changes](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=4e701482d1d7b90c358e2bd244bb71623f767120>) to the interfaces for the hash function and noted in the commit message:

The 32-bit version is more efficient (and apparently gives better hash results than the 64-bit version), so users who are only hashing a 32-bit quantity can now opt to use the 32-bit version explicitly 

So he, too, knew that there were problems with the hashing function, particularly the 64-bit version. It would be naive to think that these three are the only developers to have noticed a problem but none made the effort to fix it. In the 14 years since this code was [introduced](<http://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=e5191c50d12621e04f8bc247dcb6a0f4ad76ae26>), several people have noticed a problem and no one has bothered to fix it. This is an example of what has become known as the "[platform problem](<https://lwn.net/Articles/443531/>)".

Fortunately Torvalds was interested in the futex changes and so paid attention to the second patch set from Gleixner. He made no comment on the futex changes, but did take interest in the hash changes, [researching](<http://mid.gmane.org/CA+55aFyjYkwkTo2bNYqJ6h4mr1bbT5Vrak+EtiZmujOD-NzMOQ@mail.gmail.com>) the problem and orchestrating the fix. The key issue turned out to be the number that the input is multiplied by. Supposedly based on recommendations from Donald Knuth, this number was chosen to be a prime and to be approximately the fractional part of the [golden ratio](<https://en.wikipedia.org/wiki/Golden_ratio>) multiplied by 264 (in the 64-bit case). A prime was chosen that was "bit sparse", having long runs of ones or zeros in the binary representation.

This latter choice was guided by the desire for the multiplication to be fast. On some hardware, shifts and addition or subtraction can be faster than general multiplication and with fewer runs in the binary pattern, fewer additions or subtractions are needed. For the 64-bit case, the pattern is particularly sparse, having a run of 33 one bits.

```
/*  2^63 + 2^61 - 2^57 + 2^54 - 2^51 - 2^18 + 1 */
        #define GOLDEN_RATIO_PRIME_64 0x9e37fffffffc0001UL
```

This sparseness was the problem. Torvalds's research suggested that the primality was barely interesting and a misunderstanding. As George Spelvin later [explained](<http://mid.gmane.org/20160429233115.8864.qmail@ns.horizon.com>) it:

One thing I note is that the advice in the comments to choose a prime number is misquoting Knuth! Knuth says (vol. 3 section 6.4) the number should be *relatively* prime to the word size, which for binary computers simply means odd. 

It also appears that the golden ratio is not all that magical. It is just a convenient number that is, as Torvalds put it, ""an irrational number that is not near to 0 or 1"". The important property was to have ""roughly 50% of the bits set in a random pattern"" so the resultant mixing of bits would hide any patterns in the input. The current hash function, it seems, emphasizes exactly the wrong qualities. 

Once the problem was understood, fixing it was relatively straightforward: just pick a better multiplier with more "randomness" in the bit pattern. This was done by [approximating an irrational value](<http://mid.gmane.org/20160502102248.18755.qmail@ns.horizon.com>) and not worrying about primality or bit-sparseness. On hardware without 64-bit multiplication, the two 32-bit halves of the 64-bit input are hashed sequentially so two 32-bit multiplications are used. This doesn't return a 64-bit number, but no code ever wants more than 32 bits of hash, so not providing the full 64 is no loss.

Spelvin made good use of the opportunity provided by this need to adjust the hash functions and examined other hashing code, fixing up the `hash_mem()` and `hash_str()` function in the sunrpc code (used by `nfsd`) and suggesting improvements for some simplistic hashing used in the inode cache.

A [minimal fix](<http://mid.gmane.org/CA+55aFyPMrAWqMaA013ZQA0TULtdTXbUh4rgUurMLu40TastuA@mail.gmail.com>) has landed for Linux 4.6-rc7 with a more complete fix expected for v4.7. It is good to know that this problem with the Linux kernel platform has been addressed, but it does lead one to wonder what other problems there are that have been conveniently ignored by many of us, and just need a little light to shine on them at the right time for the problem to be quickly fixed.
