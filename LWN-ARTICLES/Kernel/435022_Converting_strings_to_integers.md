---
title: Converting strings to integers
url: https://lwn.net/Articles/435022/
date: "March 23, 2011"
category: String processing
author: "By Jonathan Corbet March 23, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 23, 2011

Kernel developers might rightly complain about being confused over which functions should be used to convert strings to integer types. Old functions like `simple_strtoul()` will silently ignore junk at the end of an integer value, so "`100xx`" successfully converts to an unsigned integer type. Alternatives like `strict_strtoul()` have been encouraged instead, but they have problems too, including the lack of overflow checks. So what's a kernel hacker to do? 

As of 2.6.39, there is a new set of string-to-integer converters which is expected to be used in preference to all others. 

  * Unsigned conversions can be done with any of `kstrtoull()`, `kstrtoul()`, `kstrtouint()`, `kstrtou64()`, `kstrtou32()`, `kstrtou16()`, or `kstrtou8()`. 

  * Conversions to signed integers can be done with `kstrtoll()`, `kstrtol()`, `kstrtoint()`, `kstrtos64()`, `kstrtos32()`, `kstrtos16()`, or `kstrtos8()`. 

All of these functions are marked `__must_check`, so callers are expected to check to ensure that the conversion happened successfully. The older functions are marked deprecated, and will eventually be removed. These new `kstrto*()` functions are now the Official Best Way To Convert Strings, so developers need wonder no longer.
