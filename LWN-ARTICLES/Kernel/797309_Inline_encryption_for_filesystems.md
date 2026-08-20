---
title: Inline encryption for filesystems
url: https://lwn.net/Articles/797309/
date: "August 27, 2019"
category: "Security-Filesystem encryption"
author: "By Jonathan Corbet August 27, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 27, 2019

The encryption of data at rest is increasingly mandatory in a wide range of settings from mobile devices to data centers. Linux has supported encryption at both the filesystem and block-storage layers for some time, but that support comes with a cost: either the CPU must encrypt and decrypt vast amounts of data moving to and from persistent storage or it must orchestrate offloading that work to a separate device. It was thus only a matter of time before ways were found to offload that overhead to the storage hardware itself. Satya Tangirala's [inline encryption patch set](<https://lwn.net/ml/linux-fsdevel/20190821075714.65140-1-satyat@google.com/>) is intended to enable the kernel to take advantage of this hardware in a general manner. 

The Linux storage stack consists of numerous layers, so it is unsurprising that an inline encryption implementation will require changes at a number of those layers. Hardware-offloaded encryption will clearly require support from the device driver to work, but the knowledge of which encryption keys to use typically comes from the filesystem running at the top of the stack. Communicating that information from the top to the bottom requires a certain amount of plumbing. 

#### Low-level support

At [the lowest level](<https://lwn.net/ml/linux-fsdevel/20190821075714.65140-2-satyat@google.com/>), device drivers that support inline encryption will have to provide an operations structure like this: 

```
struct keyslot_mgmt_ll_ops {
    	int (*keyslot_program)(void *ll_priv_data, const u8 *key,
    			       enum blk_crypto_mode_num crypto_mode,
    			       unsigned int data_unit_size,
    			       unsigned int slot);
    	int (*keyslot_evict)(void *ll_priv_data, const u8 *key,
    			     enum blk_crypto_mode_num crypto_mode,
    			     unsigned int data_unit_size,
    			     unsigned int slot);
    	bool (*crypto_mode_supported)(void *ll_priv_data,
    				      enum blk_crypto_mode_num crypto_mode,
    				      unsigned int data_unit_size);
    	int (*keyslot_find)(void *ll_priv_data, const u8 *key,
    			    enum blk_crypto_mode_num crypto_mode,
    			    unsigned int data_unit_size);
        };
```

The interface is designed around hardware that provides a fixed number of "key slots", each of which can hold a cryptographic context — the algorithm to be used, associated parameters (the block size, for example), and the key. These functions exist to program a crypto context into the hardware, remove a crypto context from the hardware, determine whether a specific context is supported, and to determine which slot, if any, is already programmed for a given context. Drivers will register this structure and provide the total number of slots available. 

Since the number of key slots provided by the hardware is fixed, it's entirely possible that there will not be enough to handle all of the I/O requests to a given device over a short period of time. That may not be a problem for a device that is occupied by a single, encrypted filesystem, but the situation could be different if there are a lot of filesystems present, or if per-directory encryption (as [supported by the ext4 filesystem](<https://lwn.net/Articles/639427/>)) is in use. So the kernel needs a way to arbitrate access to key slots, preferably one that limits the amount of (possibly expensive) slot reprogramming required. 

That arbitration begins with a "key-slot manager" abstraction. It keeps track of which slots are available at any given time and, for those that are busy, how many references (held by in-flight I/O operations) exist to each. The key-slot manager can be used to allocate slots and program encryption contexts. In normal usage, many I/O requests will use the same context, so the key-slot manager tries to keep the most frequently used keys available to the hardware and avoids programming the same key into multiple slots. 

[Moving up a layer](<https://lwn.net/ml/linux-fsdevel/20190821075714.65140-3-satyat@google.com/>), the patch set adds a new `bio_crypt_ctx` structure to the BIO structure (which represents an I/O request). When filesystem code originates a request, it can add the relevant context information, and the BIO structure will carry that information through to the block device executing the request. Adding this information requires changes in other parts of the block layer; for example, two adjacent requests cannot be merged if they are using different encryption contexts. 

#### blk-crypto

Adding key information to the BIO structure isn't quite enough, though. There is still the issue of slot management and actually programming crypto contexts into the hardware; while filesystems could arguably handle this work, it almost certainly makes sense to handle this common task within the block layer itself. But there is a further complication: the device to which a filesystem submits an I/O request may not be the device that ultimately handles that request. For example, a filesystem may be based on a RAID "device" created by the device-mapper layer; code at the filesystem level will be entirely unaware of the real physical devices that have been assembled into the virtual device it sees. So filesystems cannot directly handle details like key-slot management. 

The solution to this problem is the [blk-crypto subsystem](<https://lwn.net/ml/linux-fsdevel/20190821075714.65140-4-satyat@google.com/>), which handles the details of managing key slots and getting the key information through to the right device drivers. Whenever a BIO is submitted for execution, the blk-crypto code reacts to the presence of a crypto context by allocating a slot from the key-slot manager associated with the (immediate) target device. That implies, for reasons that we'll return to shortly, that subsystems like the device mapper must implement a simple key-slot manager, even though they perform no encryption themselves. 

Layered devices like the device manager will make any necessary modifications to BIOs they receive (including possibly splitting them into multiple BIOs), then turn around and resubmit the resulting BIOs to the lower-level devices. When this happens, the blk-crypto layer will release the key slot allocated at the intermediate level and allocate a new slot for the lower-level device, propagating the key material downward. This procedure will happen as many times as necessary until the BIO reaches a device that actually performs I/O. 

The blk-crypto code has one other useful feature: if the target device does not actually support the type of encryption requested by the filesystem (or any encryption at all), blk-crypto will fall back to using the kernel's crypto layer instead. So filesystems can request encrypted data storage without any knowledge of whether the underlying hardware supports inline encryption or not. This functionality may eventually replace the fscrypt code currently used by ext4 and F2FS to implement encryption. 

The crypto-layer fallback explains why intermediate block layers must provide their own key-slot managers. The block layer never knows whether a given BIO will be resubmitted to a lower-level device later on, so it must assume that every submission is the final one. As a result, if encryption is being performed by the kernel's crypto layer, that must happen _before_ submitting the BIO to the device; there will be no opportunity to do so afterward. The lack of a key-slot manager for any given device is a signal to the block layer that inline encryption is not supported, so the crypto-layer fallback will be performed in that case; thereafter there is no point in using inline encryption for that request even if turns out to be available. Adding a key-slot manager to layers like the device mapper is, among other things, a way of preventing the block layer from falling back too soon. 

The patch set includes a low-level implementation for [Universal Flash Storage](<https://en.wikipedia.org/wiki/Universal_Flash_Storage>) devices and upper-level support for the F2FS filesystem. As one might expect, this work is being driven by Android use cases; encrypted filesystems are important for Android devices, and offloading the actual encryption to the hardware should save both CPU time and power. 

Given the potential value of this feature, it is not surprising that there have been a few attempts to add support to the kernel. The patch set mentions three of them: a [hardware-specific solution](<https://lwn.net/ml/linux-kernel/1539789476-6098-1-git-send-email-anilc@codeaurora.org/>) that lacks generality, one that is [implemented within the crypto layer](<https://lwn.net/ml/linux-kernel/192fedba-4f7d-8230-2152-b618787fd934%40synopsys.com/>) (seen as the wrong place, since it's not a general cryptographic primitive), and one that [requires the device mapper](<https://lwn.net/ml/linux-kernel/20181211095027.GA3316%40lvlogina.cadence.com/>) to function. This implementation is an attempt to avoid those problems and provide a more general solution. It is in its fourth revision and appears to be getting close to being ready to head upstream.
