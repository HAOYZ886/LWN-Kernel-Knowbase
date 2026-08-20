---
title: "Finer-grained BPF tokens"
url: https://lwn.net/Articles/947173/
date: "October 12, 2023"
category: "BPF-Security; Releases-6.9"
author: "By Jonathan Corbet October 12, 2023"
---

By **Jonathan Corbet**  
October 12, 2023

Programs running in the BPF machine can, depending on how they are attached, perform a number of privileged operations; the ability to load and run those programs, thus, must be a privileged operation in its own right. Almost since the beginning of the extended-BPF era, developers have struggled to find a way to allow users to run the programs they need without giving away more privilege than is necessary. Earlier this year, the idea of a [BPF token](<https://lwn.net/Articles/935195/>) ran into some opposition from security-oriented developers. Andrii Nakryiko has since returned with [an updated patch set](<https://lwn.net/ml/linux-fsdevel/20230927225809.2049655-1-andrii@kernel.org/>) that significantly increases the granularity of the privileges that can be conferred with a BPF token. 

In the early days, the ability to load most BPF programs was restricted to processes with the `CAP_SYS_ADMIN` capability. That capability, though, allows a user to do far more than load BPF programs; it is essentially equivalent to full root access. In the 5.8 release, the [`CAP_BPF` capability](<https://lwn.net/Articles/822362/>) was added to regulate access to most BPF operations; other capabilities may be required as well for some specific actions. `CAP_BPF` still allows a process to do a lot of things, though, probably more than an administrator would like. 

As a result, there is a longstanding interest in finding ways to further confine what processes can do with BPF. Various approaches, including [adding authoritative hooks](<https://lwn.net/Articles/929746/>) to the Linux security module mechanism and [a special BPF device](<https://lwn.net/Articles/792124/>) have been tried and rejected. The BPF token, a sort of digital cookie conferring the right to load BPF programs, seemed like it could be headed toward a similar end. Nakryiko and other BPF developers remain convinced that the security needs for BPF are unique, though, and that a unique solution for those needs is required; they have not yet given up on the idea of a token as that solution. 

Much of the time, the answer to the limited-privilege question is to run the process needing privilege within some sort of container. [User namespaces](<https://lwn.net/Articles/532593/>) can often be used for this purpose, perhaps combined with a properly constructed mount namespace. Many of the things that BPF programs can do, such as attaching to tracepoints, are inherently global in nature, though, and cannot be contained in this way. For this reason, simply giving a process `CAP_BPF` within a user namespace is not a solution to the problem; the kernel ignores `CAP_BPF` at the namespace level. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

BPF tokens are a way to give a process within a container the equivalent of the `CAP_BPF` capability. One of the concerns expressed with the original proposal, though, was that a token might escape from the container it was intended for, causing privilege to leak into the rest of the system. In the current proposal, a BPF token is tied to both a specific instance of the BPF filesystem (which holds persistent BPF objects like maps) and a user namespace. Any given token should, as a result, be useless outside of the context it was intended for. 

The first step in enabling this functionality is to augment the BPF filesystem with a new set of mount options controlling a specific instance's interaction with BPF tokens: 

  * `delegate_cmds` lists the commands that a BPF token associated with this mount can allow. Thus, for example, a BPF filesystem could be mounted to support tokens allowing reading elements from maps but nothing else, while another could allow map creation, loading programs, or the creation of tokens. 
  * `delegate_maps` controls the types of maps that a token can enable a process to create. This option only makes sense if `BPF_MAP_CREATE` is included in `delegate_cmds`. 
  * `delegate_progs` specifies which types of programs a token can enable a process to load; `BPF_PROG_LOAD` must also be in `delegate_cmds` for any type of program loading to be allowed. 
  * `delegate_attachs` (not `attaches`, alas) controls the attach types that a token can allow — once again, if the loading of programs is allowed at all. See [this page](<https://docs.kernel.org/bpf/libbpf/program_types.html>) for a list of program and attach types. 

All of these values are bitmaps corresponding to the definitions in [`<uapi/linux/bpf.h>`](<https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/bpf.h#L121>). Thus, for example, mounting a BPF filesystem with: 

```
delegate_cmds=0x21
```

would enable `BPF_PROG_LOAD` (`0x20`) and `BPF_MAP_CREATE` (`0x01`). Nakryiko acknowledges that this is not the friendliest of interfaces, especially since the values are defined in `enum`s and the user must carefully count to find the relevant bit numbers; the ability to use symbolic names will probably appear at some point in the future. Meanwhile, the special value "`any`" is equivalent to setting all of the bits for a given option. 

Once a suitable BPF filesystem has been mounted, presumably within a container, a program with the right privileges can use the `BPF_TOKEN_CREATE` command to create a new BPF token. An open file descriptor indicating the BPF filesystem mount to use must be passed as a parameter; the resulting token will be forever associated with that BPF filesystem, which defines the operations that the token can allow. It also is attached to the user namespace associated with the BPF filesystem mount. That association prevents the token from being used outside the namespace, but has another important implication as well. 

The return value from a `BPF_TOKEN_CREATE` operation is a new file descriptor representing the token; it can then be passed to the intended user via the usual mechanisms. That user can include the token with any `bpf()` calls that it makes by putting it into the new fields added to the [the sprawling `bpf_attr` union](<https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/bpf.h#L1323>) for each command. For example, when creating a new map with `BPF_MAP_CREATE`, the token can be placed in the new `map_token_fd` field. It's worth noting that, consistent with the BPF subsystem's conventions, zero is not considered to be a legitimate file-descriptor number, and so cannot be used for the token descriptor. 

When the kernel is considering whether to allow a specific `bpf()` call to succeed, it will check for the presence of a token that allows the requested operation. Possession of the token is not sufficient to allow the operation, though; the calling process must also have the required capabilities (`CAP_BPF`, plus perhaps others for some operations). In current kernels, these capabilities must come from the global init namespace; with the patch applied, they can granted by the user namespace containing the requesting process. As a result, a container can be given the ability to execute specific BPF operations without giving that privilege to _every_ process within the container. 

Thus, a process running within a user namespace will be able to carry out BPF operations, but only if it possesses a valid BPF token, that token allows the specific operation being requested, and the process has the requisite capabilities within its user namespace. This patch series does seem to provide the way to tightly control what can be done with BPF. For now, the abilities set in a BPF filesystem mount apply to all tokens created with that filesystem. In the future, there will likely be an additional operation that allows the holder of a token to remove specific abilities, further restricting what the token allows. 

Thus far, there have been few comments on this version of the patch set, and none that question the core concept; most of the discussion has focused on the details of integrating Linux security module support. So it would seem that most of the objections to previous versions have been addressed. Should that situation hold, then the path into the mainline for this work seems reasonably clear. Token-based security mechanisms have not had a place in Linux until now, so some new ground is being broken here. That could be said to be unsurprising; BPF has been challenging established ways of doing things in Linux for some years at this point.
