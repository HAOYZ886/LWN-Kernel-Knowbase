---
title: A new device naming scheme
url: https://lwn.net/Articles/79155/
date: "April 6, 2004"
category: Device naming
author: 
---

A recent [posting](<http://lwn.net/Articles/79118/>) on linux-kernel announced the creation of a new mailing list, hosted at OSDL, for the discussion of device naming schemes. The Linux Standard Base does not currently specify device names, but its maintainers would like to change that. To that end, they are seeking input on how devices should be named on Linux systems. 

The discussion, so far, has centered around a proposal (available [in PDF format](<http://lists.osdl.org/pipermail/device_naming/attachments/20040402/1c670d44/layout-draft-0001.pdf>)) from SUSE. Its purpose is to create a set of persistent device names which will remain valid even in a hotpluggable world where the hardware configuration can change at any time. To that end, the proposal creates a version of `/dev` which is radically different from anything seen on current Linux systems. 

All of the current device names found in `/dev` are relegated to the category of "compatibility names." They will still exist, but the proposal suggests that they should be maintained by `udev`, rather than being a static part of the system. The new names, instead, will all be found in subdirectories under `/dev`. Disks will be in `/dev/disk` (with a "`k`"), and the obvious things will be found in other directories, such as `/dev/printer`, `/dev/cdrom` (these, evidently, are not "disks"), or `/dev/modem`. 

The proposal calls for another level of subdirectories before you find any actual device names. Each of the `/dev` subdirectories would be further divided into `by-path`, which names each device by how it is connected to the system; `by-serial`, which uses the device's model name and serial number; `by-uuid`, which uses a device's "universal unique identifier"; and `by-label`, which uses a device's filesystem label. Thus, a system's root partition might have all of the following names: 

  * `/dev/disk/by-path/ide-0.0-part1`
  * `/dev/disk/by-serial/ata-ST340810A-53-5BIN-part1`
  * `/dev/disk/by-label/label-ROOT`
  * `/dev/disk/by-uuid/uuid-0bee1954-b245-4df1-b2af-785fecd75b8f` 

The use of multiple names for the same device does not sit well with everybody; fears have been expressed that it could confuse users and applications which perform user-space locking by device name. The `by-path` names were received critically; since the path can change on a modern system, those names will never be persistent. There were also complaints about `by-label` and `by-uuid`; those names are meant to allow Linux systems to find and mount disks regardless of their position in the device hierarchy, but the `mount` utility already implements that functionality. 

While there have been complaints about the SUSE proposal, there have not, thus far, been a lot of alternatives put forward. Something, however, is clearly going to have to change. A Fedora Core 2 Test 2 system has almost 19,000 entries under `/dev`; this mass of names can only get larger and increasingly unmaintainable. And it fails to address the dynamic nature of devices in modern systems. Device naming looks to be an interesting issue for some time to come.
