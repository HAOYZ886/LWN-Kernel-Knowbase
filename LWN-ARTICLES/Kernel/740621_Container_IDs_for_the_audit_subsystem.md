---
title: Container IDs for the audit subsystem
url: https://lwn.net/Articles/740621/
date: "December 6, 2017"
category: Auditing
author: "By Jake Edge December 6, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
December 6, 2017

Linux containers are something of an amorphous beast, at least with respect to the kernel. There are lots of facilities that the kernel provides (namespaces, control groups, seccomp, and so on) that can be composed by user-space tools into containers of various shapes and colors; the kernel is blissfully unaware of how user space views that composition. But there is interest in having the kernel be more aware of containers and for it to be able to distinguish what user space considers to be a single container. One particular use case for the kernel managing container identifiers is the [audit subsystem](<https://github.com/linux-audit>), which needs unforgeable IDs for containers that can be associated with audit trails. 

Back in early October, Richard Guy Briggs [posted](<https://lwn.net/Articles/740730/>) the second version of his RFC for kernel container IDs that can be used by the audit subsystem. The first version was [posted](<https://lwn.net/Articles/740732/>) in mid-September, but is not the only proposal out there. David Howells [proposed](<https://lwn.net/Articles/723561/>) turning containers into full-fledged kernel objects back in May, but seemingly ran aground on objections that the proposal ""muddies the waters and makes things more brittle"", in the [words](<https://lwn.net/Articles/723565/>) of namespaces maintainer Eric W. Biederman. 

Briggs's proposal is focused on the needs of the audit subsystem, rather than trying to solve any larger problem, however. He [described](<https://lwn.net/Articles/699819/>) some of the problems for the audit subsystem in a 2016 Linux Security Summit talk. In addition, he laid out some of the requirements for container tracking in [response](<https://lwn.net/Articles/740734/>) to a [query](<https://lwn.net/Articles/740735/>) from Carlos O'Donell about the first RFC: 

  * ability to filter unwanted, irrelevant or unimportant messages before they fill queue so important messages don't get lost. This is a certification requirement. 
  * ability to make security claims about containers, require tracking of actions within those containers to ensure compliance with established security policies. 
  * ability to route messages from events to relevant audit daemon instance or host audit daemon instance or both, as required or determined by user-initiated rules 

As proposed, audit container IDs would be handled as follows. A container orchestration system would register the ID of a container (a 16-byte UUID) by writing to a special file in the `/proc` directory for the container's initial process. Briggs proposes a new capability (`CAP_CONTAINER_ADMIN`) that would be required for a process to be able to register a container ID, but no process would be able to change its own container ID even with the capability. 

Registering the container ID would associate the process ID (PID) of the first process (in the initial PID namespace) and all of that process's namespaces (using the [namespace filesystem device and inode numbers](<http://blog.man7.org/2016/12/introspecting-namespace-relationships.html>)) with the ID in an `AUDIT_CONTAINER` record that gets logged. The container IDs would then be used in various audit log messages to associate auditable events with the container that performed them. Any child processes would inherit the container ID of their parent so that all of the processes and threads in a container would be associated with its ID. If the first process has already forked or created threads, the registration would either fail or all of the child processes/threads would be associated with the ID; the right course will be determined as part of the RFC and implementation process. 

Audit events would be generated for all namespace creation and destruction operations; creation events would be associated with the container ID of the process performing the action, destruction events occur when there are no more references to a namespace, so just the device and inode of the namespace destroyed would be logged. Changes to a process's namespaces would also generate an audit event that records the new and old namespace information. 

The new capability for container IDs was one of the first things questioned about the proposal. Casey Schaufler [asked](<https://lwn.net/Articles/740743/>) how there could be a kernel container capability when the RFC clearly states that the kernel knows nothing about containers. Briggs [likened](<https://lwn.net/Articles/740744/>) container IDs to login user IDs and session IDs ""that the kernel tracks for the convenience of userspace"". He suggested that if the `CAP_CONTAINER_ADMIN` name was the problem, he would be fine with something like `CAP_AUDIT_CONTAINERID`, but that was not the core of Schaufler's [complaint](<https://lwn.net/Articles/740747/>): 

Sorry, but what aspect of the kernel security policy is this capability supposed to protect? That's what capabilities are for, not the undefined support of undefined user-space behavior. 

If it's audit behavior, you want CAP_AUDIT_CONTROL. If it's more than audit behavior you have to define what system security policy you're dealing with in order to pick the right capability. 

We get this request pretty regularly. "I need my own capability because I have a niche thing that isn't part of the system security policy but that is important!" Fit the containerID into the system security policy, and if that results in using CAP_SYS_ADMIN, oh well. 

There already are two capabilities for the audit subsystem (`CAP_AUDIT_CONTROL` and `CAP_AUDIT_WRITE`) but, as Paul Moore [explained](<https://lwn.net/Articles/740748/>), neither is quite right to govern the ability to register container IDs: 

CAP_AUDIT_WRITE exists to control which applications can submit userspace generated audit records to the kernel, CAP_AUDIT_CONTROL exists to control which applications can manage the in-kernel audit configuration (e.g. filter rules) and the current task's loginuid value. Reusing CAP_AUDIT_WRITE here would allow any application that can submit userspace audit records the ability to change the audit container ID; this would be bad, we don't allow CAP_AUDIT_WRITE to change the loginuid, it would be even worse to allow it to change the audit container ID. Reusing CAP_AUDIT_CONTROL is less worse than than CAP_AUDIT_WRITE, but it gets sticky once we get to the part where we want to auditd instances in containers, complete with their own queues, filtering rules, etc.. Perhaps we could use CAP_AUDIT_CONTROL to guard the audit container ID value, but we would always want to do that check in the init userns in order to prevent container bound processes from manipulating their own audit container ID. 

James Bottomley [suggested](<https://lwn.net/Articles/740750/>) sidestepping the capability question by making the container ID a write-once attribute; once set, nothing could change it. The idea of nested containers came up several times, though, which would require some way to change these container IDs. Bottomley suggested simply to allow appending to the container ID, so that the hierarchy is inherent in the chain of IDs. Moore [agreed](<https://lwn.net/Articles/740762/>) that write-once would work for the non-nested case: 

Richard [Briggs] and I have talked about a write once approach, but the thinking was that you may want to allow a nested container orchestrator (Why? I don't know, but people always want to do the craziest things.) and a write-once policy makes that impossible. If we punt on the nested orchestrator, I believe we can seriously think about a write-once policy to simplify things. 

But Aleksa Sarai [pointed out](<https://lwn.net/Articles/740763/>) that nested containers are a fairly common use case, for LXC system containers in particular (which will often have other container runtimes running inside them). Biederman [noted](<https://lwn.net/Articles/740764/>) that there is not, as yet, a solution for running the audit daemon in containers, so it may be premature to worry about nested container IDs at this point. 

Schaufler is [concerned](<https://lwn.net/Articles/740765/>) that adding an ID for auditing containers is heading down the wrong path. He suggested the [ptags Linux Security Module](<https://lwn.net/Articles/702639/>) as a way forward; it would allow arbitrary tags with values to be set for a process. 

Then you want Jose Bollo's PTAGS. It's insane to add yet another arbitrary ID to the task for a special purpose. Add a general tagging mechanism instead. We could add a gazillion new id's, each with [its] own capability if we head down this road. 

Moore [stressed](<https://lwn.net/Articles/740766/>) that the effort was not aimed at a more general mechanism, but simply to address the needs of the audit subsystem at this point. He said that the ID is meant to be an ""audit container ID"" and not a more general ""container ID"". Using the audit ID for other purposes risks opening up problems in other areas (such as container migration), so he and Briggs are attempting to restrict the use cases. 

We would love to have a generic kernel facility that the audit subsystem could use to identify containers, but we don't, and previous attempts have failed, so we have to create our own. We are intentionally trying to limit its scope in an attempt to limit problems. If a more general solution appears in the future I think we would make every effect to migrate to that; keeping this initial effort small should make that easier. 

At this point, there is no code on the table, it is purely a discussion on where things should go. Adding a new capability for registering these IDs seems to be a non-starter; the write-once scheme governed by one of the existing audit capabilities seems like it might plausibly pass muster. Though, as Moore said, there seems to be a bigger need here, but more general solutions have so far been hard to come by. Adding IDs willy-nilly may be suboptimal but, until something more general comes along, might just be the right way forward.
