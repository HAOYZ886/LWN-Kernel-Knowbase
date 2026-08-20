---
title: The integrity policy enforcement security module
url: https://lwn.net/Articles/817472/
date: "April 16, 2020"
category: "Security-Security modules"
author: "By Jonathan Corbet April 16, 2020"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 16, 2020

There are many ways to try to keep a system secure. One of those, often employed in embedded or other dedicated-purpose systems, is to try to ensure that only code that has been approved (by whoever holds that power over the system in question) can be executed. The secure boot mechanism, which is intended to keep a computer from booting anything but a trusted kernel, is one piece of this puzzle, but its protection only extends through the process of booting the kernel itself. Various mechanisms exist for protecting a system after it boots; a new option for this stage is the [Integrity Policy Enforcement (IPE) security module](<https://lwn.net/ml/linux-kernel/20200415162550.2324-1-deven.desai@linux.microsoft.com/>), posted by Deven Bowers. 

IPE is one of a new generation of security modules that has been enabled by [the ongoing work to implement module stacking](<https://lwn.net/Articles/804906/>). It does not attempt to provide a full security enforcement mechanism like SELinux, AppArmor, or Smack do; instead, it focuses specifically on the task of vetting attempts to execute code. And, in particular, its enforcement mechanism comes down to a simple question: does the code that the system is proposing to execute come from an appropriately signed disk volume? 

IPE is designed to work with [dm-verity](<https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/verity.html>), which provides integrity checking for block devices. Each dm-verity volume has a root hash, which is derived from the hashes of the individual blocks in that volume. Whenever blocks are read from this volume, the hashes are checked up to the root to ensure that nothing has been tampered with. Assuming everything is working as intended, the data read from a dm-verity volume is guaranteed to be the data that the creator put there and hashed, with no subsequent tampering. 

While dm-verity can be used to ensure that nobody has corrupted a disk image, there are still a couple of pieces missing when it comes to ensuring the integrity of the system as a whole. One is ensuring that the root hash for the volume is the one that the creator of the volume intended; that can be done by either storing the hash value separately or applying a cryptographic signature. Even a verified, integrity-protected volume is only of limited use, though, if the system is able to execute code that doesn't come from that volume. 

There are a few security modules that can address these problems; IPE attempts to do so in a relatively simple way. When it is active, IPE's entire purpose is to make sure that all execution is done from code found on volumes that are protected by dm-verity, and which have the appropriate hashes or signatures. To that end, it has a simple policy language that the system administrator can use to describe which executables are acceptable. 

The first line of a policy declaration is special; it provides a name and a version number: 

```
policy_name="Evil lockdown policy" policy_version=6.6.6
```

The name simply identifies the policy; the version number is used to prevent a system from being rolled back to an earlier version of the policy. IPE will, though, allow a policy to be overwritten by another with the _same_ version number, for whatever reason. 

Everything else describes a portion of the desired policy by tying an operation to an access decision. For example, this rule would allow any attempt to execute a file: 

```
op=EXECUTE action=ALLOW
```

The set of operations that can be controlled is made up of: 

  * `EXECUTE`: execute a file or load a file for execution. This includes calls to `mmap()` or `mprotect()` that would create executable memory regions. 
  * `FIRMWARE`: the loading of firmware. 
  * `KMODULE`: the loading of a kernel module with `insmod` or `modprobe`. 
  * `KEXEC_IMAGE` and `KEXEC_INITRAMFS`: booting a new kernel via the kexec mechanism; `KEXEC_INITRAMFS` controls the provision of an initramfs for the new kernel. 
  * `POLICY`: the loading of integrity-measurement policies. Note that this refers to the [IMA integrity subsystem](<https://sourceforge.net/p/linux-ima/wiki/Home/?version=8>), not IPE. 
  * `X509_CERT`: the loading of certificates into IMA. 

The choices for `action=` are just `ALLOW` or `DENY`. 

Thus far, though, we have not made any connection to verified sources of executable code. That is done by adding qualifiers to describe the provenance of specific binaries. The first step is probably to allow running code from a filesystem image that was provided by the secure boot mechanism: 

```
op=EXECUTE boot_verified=TRUE action=ALLOW
```

On the assumption that the secure boot mechanism has ensured that the initial RAM filesystem is verified, this lets the system run the programs found there. That forms part of the chain that lets the system bootstrap itself to the point of running from the real filesystem. Of course, there is no way for IPE to know for sure that secure boot was used; what this option really does is just enable trust for the initial filesystem. Once that happens, there are two more qualifiers to describe where code can (or cannot) come from: 

```
dmverity_roothash=_hash_
        dmverity_signature=TRUE|FALSE
```

The first causes the rule to apply to a dm-verity volume whose root-level hash is `_hash_`. Normally the action would be `ALLOW` to enable execution from a known-good volume. It could be set to `DENY`, though, to specifically disallow a volume that, for example, contains code with a known vulnerability. A rule with `dmverity_signature` checks whether the executable comes from a volume where the root hash has been signed by a key that appears in the kernel's trusted keyring. 

The special `DEFAULT` qualifier describes what happens when no rule matches a specific situation. For example, a line like the following would deny everything by default, and might be a logical starting point for a real-world configuration: 

```
DEFAULT action=DENY
```

If there were a need to allow, for example, the loading of kernel modules by default, the above line could be preceded by something like: 

```
DEFAULT op=KMODULE action=ALLOW
```

The first non-default rule that applies to a given situation makes the decision for the operation in question, so the ordering of rules is important. 

That is the entire policy language in the current patch set. Since it may be desirable to have IPE active before the system has reached a point where it can load a policy, an initial policy can be specified at build time with the `SECURITY_IPE_BOOT_POLICY` configuration variable. In the absence of an initial policy, IPE is disabled until a policy is loaded, which is done by the policy text to `/sys/kernel/security/ipe/new_policy`; the policy files must be signed with a key found in the trusted keyring. There can be multiple policies loaded at any given time, which is why they have names; a policy can be selected as the active policy by writing its name to the `ipe.active_policy` sysctl knob. There does not appear to be any restriction of the ability to switch between policies at run time. 

One interesting behavioral quirk is that, by default, IPE will ignore any lines in the policy file that it is not able to parse successfully. This is done in the name of compatibility, but it could also have the effect of causing policies to be changed in an undesirable way by way of a typo. The `ipe.strict_parse` sysctl knob can be used to turn such mistakes into fatal errors. 

See [this patch](<https://lwn.net/ml/linux-kernel/20200406221439.1469862-12-deven.desai@linux.microsoft.com/>) for the documentation file included with IPE. 

As is always the case with this sort of technology, IPE can be used for good or evil purposes. The most obvious use case is to lock down consumer devices, preventing their owners from making modifications. But it can also be used to, for example, ensure that a router running OpenWrt continues to run the software the owner put there. IPE, at least, is relatively simple, meaning that setting it up on a private machine is feasible without investing a great deal of time. 

As of this writing, IPE has not seen a lot of review, so it's difficult to say when, or in what form, it may be merged into the mainline. From a first reading, though, it doesn't appear that there are a whole lot of reasons to keep it out.
