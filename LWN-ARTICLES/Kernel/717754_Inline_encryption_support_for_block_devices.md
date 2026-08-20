---
title: Inline encryption support for block devices
url: https://lwn.net/Articles/717754/
date: "March 22, 2017"
category: Block layer; Security
author: "By Jake Edge March 22, 2017 LSFMM 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
March 22, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

In a combined storage and filesystem session at LSFMM 2017, Ted Ts'o led a discussion of support for inline cryptographic engines (ICEs) that are being used in mobile phones. A number of hacks have been made over the last few years by Android device makers for Linux support of these engines to encrypting filesystem data, but Ts'o would like to create something that can go into the mainline kernel. He was looking for thoughts on how to make that happen. 

Doing AES encryption on the ARM cores that are used in mobile phones is fairly power hungry, so manufacturers are increasingly turning to ICE devices to encrypt the data on the device. These ICE devices sit between the CPU and the flash storage; the CPU must provide a key ID to them in order to use them. So there is a need to tell the engine which key to use for an I/O request. In the future, Ts'o said, the keys themselves may come from a secure element, such that the CPU and kernel will not have access to them at all. 

Qualcomm has been trying to get support for ICE devices upstream for some time, but the code is "rather unspeakable". It blindly assumes an ext4 filesystem and roots through private pointers to access inode structures in order to associate key IDs with I/O requests. The Qualcomm code is not what is in the Pixel phones, he was quick to note; Qualcomm started with the Pixel code and "did horrible things to it". 

His goal is to find upstream-acceptable changes to support ICE. A "nice to have" would be a way for him to remove the hacks in the ext4 and f2fs filesystems, as well, and add a filesystem and block encryption mechanism that does not require a device-mapper layer. For the desktop and server case, having a device-mapper layer makes it easier for users, he said, but with hardware crypto, there's no reason to have one. 

Ts'o proposed adding a 32-bit key ID field to `struct bio`, which is what [Universal Flash Storage](<https://en.wikipedia.org/wiki/Universal_Flash_Storage>) (UFS) has. Key IDs are integer values that refer to keys that have been stored into "slots" in ICE device. He believes that most ICE devices will have far fewer slots than 32 bits will allow, though. 

James Bottomley suggested using the [Data Integrity Field](<https://en.wikipedia.org/wiki/Data_Integrity_Field>) and Data Integrity Extensions (DIF/DIX) support for the key IDs. Martin Petersen said there is a union that holds DIF/DIX or copy offload information; another field could be added for the key ID. Ts'o said he would look into that. 

There will also be a need for a key slot manager of some kind. Since there will be a limited number of key slots for an ICE device, there can only be that many BIOs with different key IDs in flight at any given time. So the device will need to request a key slot, which might block if there are none available. The slots will need to be reference counted; they would be incremented when a BIO with an ID is submitted and decremented when it completes. 

All of the key slot management would be hidden from the filesystem. The drivers will manage the slots, but the filesystem will need to identify the key that goes with a particular request. It is important that two BIOs with different keys do not get merged. David Howells asked about superblock encryption and whether `mount()` needs to know about keys, but Ts'o said that the metadata for the ext4 and f2fs filesystems is not encrypted on Android devices. There is some rough prototype code that Michael Halcrow has been working on that should come out soon, Ts'o said. 

In something of a side note, he also mentioned that right now filesystem encryption on desktops or servers uses a per-process or per-session key ring. Users can set and remove their own keys from those rings, but that doesn't work for hardware devices because there is no concept of a key owner. Once a key gets into an ICE device, there are no further checks and anyone can use the key. It is the host operating system that allows or prevents access to files using the Unix permissions. 

It would be useful to have a kind of global key ring for software crypto that could be used like an ICE device, he said. Keys would be added or removed only by root, but once they are added, those keys can be used by anyone on the system. Someone in the audience asked about containers where there may be multiple "root" users due to user namespaces. Ts'o said he hadn't thought about it. Someone suggested tying the key ring to the user namespace where they were created, so a global key ring created by root in a container would only be accessible to other users in that container/namespace.
