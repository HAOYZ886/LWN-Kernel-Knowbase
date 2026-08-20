---
title: The lightweight auditing framework
url: https://lwn.net/Articles/79326/
date: "April 7, 2004"
category: Auditing; SELinux
author: 
---

One of the patches in Andrew Morton's "merge candidate" tree is the lightweight audit framework. This patch, written by Rik Faith, is intended to be a way for the kernel to get various types of audit information out to user space without slowing things down, especially when auditing is not being used. The framework is meant to serve as a complement to SELinux; it is already being shipped as a part of the Fedora Core 2 test 2 kernel. 

There are two kernel-side components to the audit code. The first is a generic mechanism for creating audit records and communicating with user space. All of that communication is performed via netlink sockets; there are no new system calls added as part of the audit framework. Essentially, a user-space process creates a `NETLINK_AUDIT` socket, writes `audit_request` structures it, and reads back `audit_reply` structures in return. 

The generic part of the audit mechanism can control whether auditing is enabled at all, perform rate limiting of messages, and handle a few other tasks. On the kernel side, it provides a `printk()`-like mechanism for sending messages to user space. This code also implements a user-specified policy on what happens if memory is not available for auditing; truly paranoid administrators can request that the kernel panic in such situations. 

The audit patch includes some SELinux tweaks to make it use the audit functions rather than `printk()` when it has something to log. 

The audit logging code expects an audit daemon to be running to accept messages via the netlink socket. Code for an example daemon is available [in Rik's Red Hat web area](<http://people.redhat.com/faith/audit/>). Should there be no daemon running, log messages are simply passed to `printk()` instead. 

In addition to the generic support code, the audit patch includes a mechanism for auditing system calls. One gets the sense that this was the real purpose for the patch. System call auditing is off by default, but a suitably privileged user-space process can turn it on and load a whole set of rules describing what should be logged. Rules can test on various attributes of the calling process, including its process ID, user and group ID (both "real" and "effective"), etc. Rules can also be set to fire on accesses to particular devices or files. Finally, there are also tests on specific system call arguments, whether the call succeeds, or for a specific return value. 

Included with the audit daemon is an `auditctl` utility which can be used for setting and tweaking rules. 

The audit mechanism will give system administrators a new tool for looking at what is going on between user space and the kernel. With the addition of some user-space utilities, it could become a powerful facility for tracking down system problems and security issues - or for any number of big-brotherish applications. Expect to see it in 2.6.6.
