---
title: "The word-at-a-time interface"
url: https://lwn.net/Articles/501492/
date: "June 12, 2012"
category: "String processing; Word-at-a-time"
author: "By Jonathan Corbet June 12, 2012"
---

By **Jonathan Corbet**  
June 12, 2012

While the kernel is a crucial part of almost any job one might want to do with a computer, it is rarely the kernel itself that gets the interesting work done. More often, it appears as overhead that takes system resources away from the applications the user actually wants to run. So it makes sense to optimize kernel operations to the greatest extent possible, especially when those operations are carried out frequently on performance-critical paths. The "word at a time" interface, optimized and made generic for the 3.5 release, is a good example of how far those optimization efforts can go. 

The kernel does a lot of string processing, especially (but not exclusively) when working with file path names. It is often necessary to know the length of a name or path component. When confronted with such a task, a C programmer would typically code a loop iterating through the string one character at a time. But, given enough strings, the per-character loop overhead starts to get expensive. It turns out that, with enough bit-level trickery, much of that overhead can be dealt with by working through string data one 32-bit or 64-bit word at a time. The "word at a time" API makes that sort of processing possible—but with a certain complexity cost. 

#### The API

Code wanting to use this interface should include `<asm/word-at-a-time.h>`. A few functions are defined therein, the first being `has_zero()`: 

```
unsigned long has_zero(unsigned long value, unsigned long *bits, 
    			   const struct word_at_a_time *constants);
```

From one point of view, `has_zero()` is a simple boolean function that returns true if `value` contains a zero byte. But what are the other two parameters? Let's start with the `constants` value, which must simply be set to a value defined in the header file: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

```
const struct word_at_a_time constants = WORD_AT_A_TIME_CONSTANTS;
```

As will be described in more detail below, this structure contains some useful constant values. The structure is small and the contents are architecture-dependent, so it was evidently deemed unnecessary to create a single, globally-accessible copy. 

The `bits` parameter, instead, is a place where `has_zero()` can stash temporary data that will be useful to the remaining functions in the API. Those functions are: 

```
unsigned long prep_zero_mask(unsigned long value, unsigned long bits,
        				 const struct word_at_a_time *constants);
        unsigned long create_zero_mask(unsigned long bits);
        unsigned long find_zero(unsigned long mask);
```

Once `has_zero()` has identified a word containing a zero byte, all three of these functions must be used to determine _which_ byte contains the zero value. The usual calling sequence looks something like this: 

```
if (has_zero(value, &bits, &constants)) {
            bits = prep_zero_mask(value, bits, &constants);
        	bits = create_zero_mask(bits);
        	zero_byte = find_zero(bits);
    	/* ... */
```

In other words, `prep_zero_mask()` and `create_zero_mask()` both take the `bits` value first created by `has_zero()` and rework it to the point that `find_zero()` can produce the offset of the first zero byte in the word. 

This may seem like a lot of work to do, but there is a reason for it. The functionality split allows different architectures to provide optimized functions for each part of the job. But there is another interesting bit of subtlety here: it is possible to perform a logical OR of two different `bits` values from two calls to `prep_zero_mask()`. The function `hash_name()` in `fs/namei.c` uses this feature to search for either a zero byte or one containing a slash—the string it is looking at ends either with a null byte or the beginning of the next component in the path name. The kernel spends a lot of time processing path names, so this operation is worth optimizing in this way. 

There is one other little detail to be kept in mind: the string might not start at the beginning of a word. Managing unaligned strings adds a bit more complexity to the task; the curious can look at `lib/strnlen_user.c` for one example of how these strings are handled. All told, using this interface adds enough complexity that it is probably almost never worthwhile. In that rare case where a per-character loop is too expensive, though, word-at-a-time access can help. 

#### How it works

The x86 version of this API can be found in `arch/x86/include/asm/word-at-a-time.h`; one might be forgiven for thinking that parts of it came from the obfuscated C contest. It starts by defining the above-mentioned constants: 

```
struct word_at_a_time {
    	const unsigned long one_bits, high_bits;
        };
    
        #define WORD_AT_A_TIME_CONSTANTS { REPEAT_BYTE(0x01), REPEAT_BYTE(0x80) }
```

`REPEAT_BYTE()` is a macro (defined in `<linux/kernel.h>`) that fills a word with copies of the given byte value. So, on a 32-bit machine, `one_bits` will be initialized to `0x01010101`, and `high_bits` will be `0x80808080`; 64-bit machines will get the same pattern, but twice as long. 

After that, `has_zero()` is defined as: 

```
static inline unsigned long has_zero(unsigned long a, unsigned long *bits, 
        					 const struct word_at_a_time *c)
        {
    	unsigned long mask = ((a - c->one_bits) & ~a) & c->high_bits;
    	*bits = mask;
    	return mask;
        }
```

In English, the code subtracts one from every byte, masks out all of the bits that were set in the original value, then masks everything but the highest bit in every byte. If one thinks of each byte as an independent value, the high bit can be thought of as the sign bit. Subtracting one from a value will only cause the sign bit to change from zero to one if the bytes's value was zero before. So this series of operations will cause the highest bit to be set in any byte whose value was zero before. (In truth, the bytes are not independent, and borrowing will cause different results after the first zero byte, but only the first one is of interest so that is unimportant). 

In the x86 implementation, `prep_zero_mask()` does nothing and will be optimized out by the compiler. That is not true of `create_zero_mask()`, though: 

```
static inline unsigned long create_zero_mask(unsigned long bits)
        {
    	bits = (bits - 1) & ~bits;
    	return bits >> 7;
        }
```

The subtraction will cause all bits up to the first set bit to be set to one; all of the other bits are then masked out and the result is right-shifted. Thereafter, all bytes prior to the first zero byte (in the original value) will be set to `0xff`. All that's left, now, is to figure out how many of those fully-populated bytes there are. The code that does this is not entirely straightforward; it is the result of [a request Linus posted on Google+](<https://plus.google.com/102150693225130002912/posts/7bKRjV92snH>) in March. For 32-bit machines, `find_zero()` comes down to this code: 

```
long a = (0x0ff0001+mask) >> 23;
        /* Fix the 1 for 00 case */
        return a & mask;
```

On 64-bit systems, instead, it looks like: 

```
return mask*0x0001020304050608ul >> 56;
```

Either way, the effect is to produce a number that is the byte offset of the first zero. 

This API is relatively new, having been first added (for x86 only) in the 3.4 development cycle. In 3.5, it was substantially reworked and made more widely applicable. There are specific implementations for x86 and powerpc (the latter uses a "count leading zeroes" instruction to speed things up); there is also a "generic" version that really only works properly for big-endian architectures. That is enough, though, for a number of architectures to make use of the capability. The resulting microsecond-sized time savings may not seem like much, but, multiplied across all of the string operations the kernel does, it adds up to a significant improvement.
