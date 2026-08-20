---
title: "No more 24-bit atomic_t"
url: https://lwn.net/Articles/71732/
date: "February 18, 2004"
category: atomic t
author: 
---

The `atomic_t` type in the Linux kernel is a simple integer variable with a set of operations which are guaranteed to be atomic without the need for explicit locking. For years, `atomic_t` variables have operated under the constraint that they can be expected to hold no more than 24 bits; this limitation was forced by the Sparc32 architecture, which used the other eight bits to implement the atomic operations. 

As of 2.6.3, this limitation no longer holds. [This patch](<https://lwn.net/Articles/71451/>) by Keith M Wesolowski has changed the Sparc32 implementation to a version (taken from the PA-RISC architecture) which provides full 32-bit atomic variables. 

The new implementation works by creating a small array (four entries) of spinlocks. When an operation is to be performed on an atomic variable, one of those spinlocks is chosen by a hash function; the code holds the given lock while manipulating the variable. The result is proper locking for atomic operations without doubling the size of every `atomic_t` in the system. The patch was quickly picked up and merged, and kernel programmers have one less strange limitation to worry about.
