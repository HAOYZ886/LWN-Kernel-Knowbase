---
title: Rule set based access control
url: https://lwn.net/Articles/153322/
date: "September 27, 2005"
category: "RSBAC; Security-Security technologies"
author: 
---

SELinux has become, to many, _the_ mechanism for high-security Linux deployments. The SELinux framework is considered sufficiently powerful, flexible, and universal that some developers have contemplated removing the Linux security module (LSM) interface altogether. When SELinux does everything, why have hooks for anything else? The fact of the matter, however, is that SELinux is not the only high-security approach out there. On September 27, [version 1.2.5](<http://lwn.net/Articles/153289/>) of the [Rule Set Based Access Control (RSBAC)](<http://rsbac.org/>) patch was released. RSBAC has been around for several years, but it has never quite achieved the prominence of SELinux. 

Like SELinux, RSBAC inserts hooks throughout the kernel source. RSBAC does not use the LSM framework, however. [This page](<http://www.rsbac.org/documentation/why_rsbac_does_not_use_lsm>) explains why; in short, the RSBAC developer (Amon Ott) does not like how LSM exposes kernel internals to security modules, and the LSM hooks are not nearly extensive enough for RSBAC. In fact, RSBAC adds hooks in many places (individual device drivers, for example) where LSM does not tread. RSBAC hooks can also change system state in ways not allowed with the LSM framework. 

With the hooks in place, RSBAC allows for several different access control regimes, all of which can be mixed and matched as desired. Available options include: 

  * Authenticated user: essentially a list of user IDs which may be assumed by each process on the system. This module is required by most other RSBAC security schemes. 

  * [User management](<http://www.rsbac.org/documentation/different_models/um>): a replacement for the PAM and shadow mechanisms which moves most of the user and group management tasks into the kernel. 

  * [Role compatibility](<http://www.rsbac.org/documentation/different_models/rc>): assigns roles to users and programs, and ensures that they match at run time. 

  * [Access control lists](<http://www.rsbac.org/documentation/different_models/acl>): a variant of file ACLs which can take additional RSBAC features (such as roles) into account. 

  * [Mandatory access control](<http://www.rsbac.org/documentation/different_models/mac>): assigns security levels to processes and objects, and prevents access between different levels. 

  * [Dazuko](<http://www.rsbac.org/documentation/different_models/daz>): a specialized interface for virus scanning applications. Dazuko creates a special purpose device which can be used to intercept file accesses; malware scans can then be performed before the access is allowed to succeed. There is a ClamAV interface to Dazuko. 

There are several other models available, see [the RSBAC models page](<http://www.rsbac.org/documentation/different_models>) for the full list. One thing that should be clear is that the RSBAC framework has been used to implement a wide variety of access control mechanisms. The project's long history suggests a stable user base, and RSBAC has been adopted by some distributions (including the [Adamantix](<http://adamantix.org/>) (formerly "Trusted Debian") and [Hardened Gentoo](<http://www.gentoo.org/proj/en/hardened/>) projects). The non-LSM approach seems likely to keep RSBAC out of the mainline kernel indefinitely (nobody is even proposing merging it), but RSBAC appears to be a viable option regardless.
