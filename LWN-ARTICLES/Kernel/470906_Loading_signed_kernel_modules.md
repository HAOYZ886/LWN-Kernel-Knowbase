---
title: Loading signed kernel modules
url: https://lwn.net/Articles/470906/
date: "December 7, 2011"
category: "Modules-Signed; Security-Security technologies"
author: "By Jake Edge December 7, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
December 7, 2011

Inserting loadable modules into a running kernel is clearly a convenient feature. It allows distributions to have relatively small kernels while still supporting a wide variety of hardware and use-cases, but it can also allow unwanted modules to be loaded in a way that may not be all that easy to detect. Those modules could simply be binary-only drivers that the distribution or service provider doesn't want to support--or they could be some kind of malware. A recently posted patch set will help to avoid those problems by giving the option of building a kernel that will only allow modules that have been cryptographically signed to be loaded, or to simply detect the presence of unsigned modules. 

David Howells posted the [patches](<https://lwn.net/Articles/470435/>), which are based on code that has been running in Fedora and RHEL kernels for years. Therefore, it should have been ""thoroughly tested"" Howells said in the final [patch](<https://lwn.net/Articles/471013/>) in the series (which also contains the module-signing.txt kernel Documentation file). The patch allows for signing modules using the RSA signature algorithm with any of the SHA family of hash algorithms. 

The basic idea is that public keys can be built into the kernel and, if `CONFIG_MODULE_SIG` is enabled, used to check the validity of modules before they are loaded. If `CONFIG_MODULE_SIG_FORCE` is enabled at compile time (or `enforcemodulesig=1` is passed on the kernel command line), only those modules that can be verified with one of the public keys built into the kernel will be loaded. If the "force" option is not used, unsigned modules will still be loaded. In either case, modules with corrupt or incorrect signatures, or those that are signed with a key that is not on the keyring, will be rejected. 

In order to make that work, there needs to be a way to build signed modules. That is done by creating public and private keys in the top-level kernel directory (in `kernel.pub` and `kernel.sec` by default). The public key will be processed with the kernel's `bin2c` utility and written as `crypto/signature/key.h`. The public and secret key will then be used with GNU Privacy Guard (GPG) to sign the modules automatically as they are built. There are several options that can be passed on the make command line to govern the location of the key files as well as options to pass to `gpg`. 

In addition, the modules can be stripped for inclusion into initial ramdisk images and debuginfo can be included in a separate ELF section that is not included in the signature calculation. That means that all of the variants of a particular module can share a single signature that is stored in the module itself (in the `.module_sig` ELF section). In addition, the output of `/proc/modules` has been changed to add a "U" to unsigned modules so that they can be detected. 

The patch also modifies the kernel's crypto subsystem to allow for the new key type and to add an RSA signature verification algorithm. That requires using the multi-precision integer (MPI) library from GPG, which was reworked from the Red Hat version for kernel inclusion by Dmitry Kasatkin. That code is already in the security tree in order to support the Extended Verification Module (EVM) digital signature extension. It also requires a minimal parser for the OpenPGP format (which is the kind of keys and signatures that GPG generates). 

There is also something of a chicken-and-egg problem. Many distributions have crypto and hash algorithms built as modules, but the RSA algorithm and whichever hash is being used to generate the signatures needs to be present or an enforcing kernel won't be able to load any modules at all. For that reason, the patches ensure that if module signatures are selected at compile time, the RSA and chosen hash algorithms are not built as modules themselves. Modules are then loaded in the usual way, with `insmod`, but the signature will be checked by suitably configured kernels. 

There has been relatively little discussion or complaints about the patches in the three revisions that have been posted since Howells began the process in late November. H. Peter Anvin is [concerned](<https://lwn.net/Articles/471023/>) about adding a OpenPGP parser to the kernel, and Howells was quick to [point out](<https://lwn.net/Articles/471024/>) that the parser is just the minimal amount needed to pull keys and signatures out of OpenPGP-formatted data. Both Anvin and James Morris were unconvinced about the need for supporting the (now deprecated) DSA signature algorithm and Howells has pulled that code out in the most recent revision. 

This is not the first time Howells has proposed these (or similar) changes as those [efforts](<https://lwn.net/Articles/92617/>) stretch back to (at least) 2004. He has now [requested](<https://lwn.net/Articles/471006/>) that the code be included into the Morris's security tree. If that happens, and no major complaints arise, we could potentially see signed module support in Linux 3.3. 

While it is a useful feature, particularly for those trying to support Linux kernels _without_ random drivers from who-knows-where, it only places another set of hurdles in front of malware authors. Since root privileges are required to load modules in the first place, a malware author will only need to find a way to insert code into the running kernel without using the module loading facility. Once upon a time, `/dev/kmem` could be used for that, but many distribution kernels don't support it any more. Prior to the advent of `CONFIG_STRICT_DEVMEM`, `/dev/mem` would have provided another way, but distributions are generally enabling that option as well. Exploiting some kind of kernel bug is the most probable route for these root-privileged attackers, but is certainly a more fragile approach than simply inserting a module. 

Another potential use (or abuse depending on perspective) of the feature is for device makers or distributions to lock down their kernels. That would leave users who wish to add functionality (or remove anti-features) in the same place as the malware author: looking for a kernel bug to exploit. Of course, some users may just find a way to replace the kernel entirely in that scenario.
