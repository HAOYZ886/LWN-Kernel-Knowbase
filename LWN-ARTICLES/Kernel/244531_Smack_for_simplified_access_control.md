---
title: Smack for simplified access control
url: https://lwn.net/Articles/244531/
date: "August 8, 2007"
category: "Modules-Security modules; Security-Security modules"
author: "By Jake Edge August 8, 2007"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
August 8, 2007

SELinux provides a comprehensive security solution for Linux, but it is large and complex. A much simpler approach is taken by the Simplified Mandatory Access Control Kernel (Smack), a [patch](<http://lwn.net/Articles/243921/>) posted to linux-kernel by Casey Schaufler. Like SELinux, Smack implements [Mandatory Access Control](<http://en.wikipedia.org/wiki/Mandatory_access_control>) (MAC), but it purposely leaves out the role based access control and type enforcement that are major parts of SELinux. Smack is geared towards solving smaller security problems than SELinux, requiring much less configuration and very little application support.

Smack allows an administrator to define labels, 1-7 characters in length, for kernel objects. Labels on objects are compared with the labels of a task that tries to access them. By default, access is only allowed if the labels match. There are a set of Smack-reserved labels that follow a different set of rules, which allows most system objects and processes to be unaffected by Smack restrictions. By default, Smack does not get in the way of the OS, allowing the administrator to concentrate on just the users and processes they want to secure. 

Smack uses filesystem [extended attributes](<http://en.wikipedia.org/wiki/Extended_file_attributes>) to store labels on files; administrators set the labels using the `attr` command. The `security.SMACK64` attribute is used to store the Smack label on each file, so setting `/dev/null` to have the Smack-reserved "star" label would look like: 

```
attr -S -s SMACK64 -V '*' /dev/null
```

For networks, [NetLabel](<http://lwn.net/Articles/204905/>) is used to set CIPSO labels and domains of interpretation for sockets, allowing Smack systems to interoperate in those strictly controlled networking environments. 

An administrator can add rules, but there is no support for wildcards or regular expressions; each rule must specify a subject label, object label and the access allowed explicitly. The access types are much like the traditional UNIX `rwx` bits, with the addition of an `a` bit for append. For configuration, Smack uses the SELinux technique of defining a filesystem that can be mounted, `smackfs`. Typically, it will be mounted as `/smack`, providing various files that can be read or written, to govern Smack operation. For example, Smack access rules are written to `/smack/load`; to change rules, one just writes a new set of access permissions for the subject-object pair. 

An example, one of several provided in the patch announcement, uses the standard security levels for government documents. Smack labels are defined for each level: `Unclass` for unclassified, `C` for classified, `S` for secret, and `TS` for top secret. Then, with a handful of rules: 

```
C        Unclass       rx
            S        C             rx
            S        Unclass       rx
            TS       S             rx
            TS       C             rx
            TS       Unclass       rx
```

the traditional hierarchy of access is defined. Because of the Smack defaults, `Unclass` will only be able to access data with that same label, whereas because of the rules above, `TS` can access `S`, `C` and `Unclass` data. 

Note that there is no transitivity in Smack rules, just because `S` can access `C` and `TS` can access `S`, that does not mean that `TS` can access `C`. That rule must be explicitly given. Also, because no write permissions have been given, tasks at each level can only write data with their own label. So secret tasks write secret data and so on. Files will inherit the label of the task that creates them, with Smack ensuring that the filesystem attribute is set. They will retain that label unless it is explicitly reset by an administrator using the `attr` command. 

A patched version of `sshd` is [available from Schaufler's homepage](<http://www.schaufler-ca.com/>) which allows an administrator to assign labels to users. Those labels get set on the user's shell and terminal device as they log into the system, forcing the user to follow the rules established for their label. A patched version of `ls` is also available so that it can display the labels associated with files. 

Smack is useful for limiting user and specific process access to various resources, it is not meant to be as general purpose as SELinux. Constructing a set of Smack labels and rules governing system processes, network services and the like, to restrict their access as SELinux does, would be impossible. For administrators needing to secure those services, SELinux is probably a better tool, but for simple compartmentalization, Smack may well suffice.
