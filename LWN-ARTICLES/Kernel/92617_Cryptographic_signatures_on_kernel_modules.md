---
title: Cryptographic signatures on kernel modules
url: https://lwn.net/Articles/92617/
date: "July 7, 2004"
category: "Cryptography; Modules-Signed; Security-Security technologies"
author: 
---

The latest Fedora Rawhide kernels come with an interesting feature: the ability to enforce cryptographic signatures on loadable modules. This capability has a few uses: 

  * Preventing the kernel from loading modules which have somehow been corrupted. 

  * Making it harder for an attacker to install a rootkit on a compromised system. 

  * Enabling vendors of enterprise Linux distributions to block the loading of unapproved modules into stock kernels. (It should be noted that, at this point, no vendor has indicated any plans to restrict module loading in this way.) 

The code which handles signed modules was originally written by Greg Kroah-Hartman; it has subsequently been fixed up in various ways by David Howells. Greg wrote [a Linux Journal article](<http://www.linuxjournal.com/article.php?sid=7130>) about his work back in January. 

The signature code works by looking at the most interesting ELF sections within a module file: the `.text` (program code) and `.data` (initialized data) areas. When the module is built, a script uses the `objdump` utility to extract those sections; the result can be fed to `gpg` to generate a signature. That signature is then patched into the module as yet another section, called `module_sig`. Overall, adding signatures is a relatively small change to the module build process. 

The signatures are not much use, however, if nobody checks them; implementing that check within the kernel is a somewhat larger business. The 2.6 kernel includes a whole cryptographic subsystem, but that code is oriented toward the needs of networking and encrypted filesystems. Verifying module signatures using public keys was not one of the objectives when the crypto API was added. To support this task, several thousand lines of code must be added to the kernel; they perform arbitrary-precision integer arithmetic (this code came directly from GnuPG), DSA signature verification (also from GnuPG), simple in-kernel key management, and the code to actually verify module data against signatures. 

As things stand in the patch currently, any public keys used to verify modules are built directly into the kernel itself. Being able to add a site-specific key at run time would be a convenient feature, but it would also defeat the purpose of this whole exercise. Any attacker who is in a position to load malevolent modules could just load a new key first, thus circumventing the signature verification. Even as things stand, a kernel using signature verification should be set up to not allow overwriting of in-kernel key data by way of `/dev/kmem` and such. 

With all that infrastructure in place, a relatively small set of patches makes the module loader actually verify signatures. Once again, the interesting sections are stripped out, and a checksum is generated with the SHA1 algorithm. If the signature in the module (1) can be decrypted with a public key contained within the kernel, and (2) contains the same checksum, the module checks out and can be loaded. 

In the code, one can see the traces of a kernel developer encountering an interesting problem. In many systems, the SHA1 transform code is kept in a loadable module. The module loader, when it attempts to verify the signature of a different module, could well force the kernel to try loading the SHA1 module. The module code, however, takes the `module_mutex` semaphore very early in the process; the recursive attempt will thus simply deadlock the whole thing. To avoid this problem, the crypto API was enhanced with a `crypto_alloc_tfm2()` function which can be instructed to not load any modules while setting itself up. The SHA1 code will have to be linked directly into the kernel if it is used for module verification. 

Rawhide kernels come configured to verify any signatures found in modules, but they will also happily load modules with no signature at all. There is a configuration option which tightens things up, however, so that only signed modules will be accepted. One wonders how much a proprietary module vendor might pay to have their public key included in a distributor's stock kernels once that option is turned on.
