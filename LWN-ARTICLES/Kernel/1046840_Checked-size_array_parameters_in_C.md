---
title: "Checked-size array parameters in C"
url: https://lwn.net/Articles/1046840/
date: "December 1, 2025"
category: "C language safety; Releases-6.19"
author: "By Jonathan Corbet December 1, 2025"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 1, 2025

There are many possible programmer mistakes that are not caught by the minimal checks specified by the C language; among those is passing an array of the wrong size to a function. A recent attempt to add some safety around array parameters within the crypto layer involved the use of some clever tricks, but it turns out that clever tricks are unnecessary in this case. There is an obscure C feature that can cause this checking to happen, and it is already in use in a few places within the kernel. 

The discussion started when Ard Biesheuvel [sought](<https://lwn.net/ml/all/20251114180706.318152-2-ardb+git@google.com>) to improve the safety of the poetically named function [`xchacha20poly1305_encrypt()`](<https://elixir.bootlin.com/linux/v6.17.8/source/lib/crypto/chacha20poly1305.c#L112>): 

```
void xchacha20poly1305_encrypt(u8 *dst, const u8 *src, const size_t src_len,
    			           const u8 *ad, const size_t ad_len,
    			       	   const u8 nonce[XCHACHA20POLY1305_NONCE_SIZE],
    			       	   const u8 key[CHACHA20POLY1305_KEY_SIZE]);
```

A potential problem with this function is that it takes as parameters several pointers to arrays of type `u8`. As Biesheuvel pointed out, the size of the `nonce` and `key` arrays is not checked by the compiler, even though it is clearly specified in the function prototype. That makes it easy to, for example, give the parameters in the wrong order. The resulting vulnerabilities are generally not the outcome developers have in mind when they write cryptographic code. 

Biesheuvel suggested that it was possible to write the prototype this way instead (differences shown in bold): 

```
void xchacha20poly1305_encrypt(u8 *dst, const u8 *src, const size_t src_len,
    			           const u8 *ad, const size_t ad_len,
    			       	   const u8 **(*nonce)**[XCHACHA20POLY1305_NONCE_SIZE],
    			       	   const u8 **(*key)**[CHACHA20POLY1305_KEY_SIZE]);
```

The types of the last two arguments have changed; there is a new level of pointer indirection, with the argument being a pointer to an array of a given size. Callers must change their calls by adding an additional `&` operator to obtain the desired pointer type, but the address that is passed is the same. In this case, though, the compiler will check the sizes of the array passed, and will now catch a reordering of the arguments to the function. 

Jason Donenfeld [was interested](<https://lwn.net/ml/all/aRePu_IMV5G76kHK@zx2c4.com>) by the idea, but he knew of an arguably more straightforward way to address this problem. It seems that, buried deep within the C standard, is a strange usage of the `static` keyword, making it possible to write the prototype as: 

```
void xchacha20poly1305_encrypt(u8 *dst, const u8 *src, const size_t src_len,
    			           const u8 *ad, const size_t ad_len,
    			       	   const u8 nonce[**static** XCHACHA20POLY1305_NONCE_SIZE],
    			       	   const u8 key[**static** CHACHA20POLY1305_KEY_SIZE]);
```

This, too, will cause the compiler to check the sizes of the arrays, and it does not require changes on the caller side. Unlike the pointer trick, which requires an exact match on the array size, use of `static` will only generate a warning if the passed-in array is too small. So it will not catch all mistakes, though it is sufficient to prevent memory-safety and swapped-argument problems. 

Eric Biggers [pointed out](<https://lwn.net/ml/all/20251115021430.GA2148@sol>) that GCC can often generate "array too small" warnings even without `static`, but that the kernel currently disables those warnings; that would suppress them when `static` is used as well. (The warning was [disabled in 6.8](<https://git.kernel.org/linus/021533194476>) due to false positives.) But he thought that adding `static` was worthwhile to get the warnings in Clang — if Linus Torvalds would be willing to accept use of ""this relatively obscure feature of C"". 

Torvalds, as it turns out, [has no objection](<https://lwn.net/ml/all/CAHk-=wj6J5L5Y+oHc-i9BrDONpSbtt=iEemcyUm3dYnZ3pXxxg@mail.gmail.com>) to this usage; he likes the feature, if not the way it was designed: 

> The main issue with the whole 'static' thing is just that the syntax is such a horrible hack, where people obviously picked an existing keyword that made absolutely no sense, but was also guaranteed to have no backwards compatibility issues. 

He pointed out that there are a number of places in the kernel that are already using it; for a simple example, see [`getconsxy()`](<https://elixir.bootlin.com/linux/v6.17.8/source/drivers/tty/vt/vt.c#L4981>) in the virtual-terminal driver. He suggested perhaps hiding it with a macro like `min_array_size()` just to make the usage more obvious, but didn't seem convinced that it was necessary. Donenfeld [followed up](<https://lwn.net/ml/all/20251118170240.689299-1-Jason@zx2c4.com>) with a patch to that effect a few days later, but then pivoted to [an `at_least` marker](<https://lwn.net/ml/all/20251122025510.1625066-4-Jason@zx2c4.com/>) instead. 

A lot of work has gone into the kernel to make its use of C as safe as possible, but that does not mean that the low-hanging fruit has all been picked. The language has features, such as `static` when used to define formal array parameters, that can improve safety, but which are not generally known and not often used. In this particular case, though, it would not be surprising to see this ""horrible hack"" come into wider use in the near future.
