---
title: Using the firmware loader for static data
url: https://lwn.net/Articles/284104/
date: "May 28, 2008"
category: Firmware
author: "By Jake Edge May 28, 2008"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
May 28, 2008

Some device drivers need firmware to load into the hardware at initialization time. The kernel firmware loader interface exists to support that functionality, but it requires help from user space which may not be available in all environments. David Woodhouse has proposed a patch that would eliminate that requirement so that more drivers can use the firmware loader rather than craft their own solution. 

Embedded devices will be one of the main users of this ability. Many of those do not have a user space filesystem available at boot time--via initrd or initramfs--but they still need to access firmware images to download to peripherals. The new `request_firmware()` implementation would allow those devices to link the firmware into the kernel while still using the kernel firmware infrastructure. 

Woodhouse has an excellent summary of what he is trying to do in the [patch posting](<http://lwn.net/Articles/283759/>): 

Some drivers have their own hacks to bypass the kernel's firmware loader and build their firmware into the kernel; this renders those unnecessary. 

Other drivers don't use the firmware loader at all, because they always want the firmware to be available. This allows them to start using the firmware loader. 

A third set of drivers already use the firmware loader, but can't be used without help from userspace, which sometimes requires an initrd. This allows them to work in a static kernel. 

A driver that has static firmware data, declares it using: 

```
DECLARE_BUILTIN_FIRMWARE("firmware_name", blob);
```

The `firmware_name` is used as a key to find the specific firmware when `request_firmware()` is called. `blob` is a pointer to the actual code. The declaration adds the firmware to the end of an array holding `struct builtin_fw` elements, which look like this: 

```
struct builtin_fw {
                char *name;
                void *data;
                unsigned long size;
        };
```

When a call is made to `request_firmware()`, the new code linearly searches the array for a matching key before calling out to user space. This allows any statically created firmware blobs to take precedence over those in the filesystem. Whichever is found is returned. 

There seemed to be strong agreement that Woodhouse's approach was the right way to go. His original implementation copied the firmware blob before returning it to a `request_firmware()` caller which required a `vmalloc()`--a waste of precious memory on embedded devices. Woodhouse was concerned that some drivers might modify the firmware before loading it into the device. Once he started looking, he found examples of that, but instead of penalizing all devices, he changed the firmware data returned in a `struct firmware` to be constant, resulting in the following structure: 

```
struct firmware {
                size_t size;
                const u8 *data;
        };
```

This constitutes an API change for anyone using the `request_firmware()` interface. In-tree drivers have been modified by Woodhouse appropriately, but out-of-tree drivers need to be aware of the change. Any driver that needs to modify the data must make a copy for themselves. 

Another feature that would be useful for memory-constrained devices is compression of the firmware in the kernel image. This is on Woodhouse's [radar](<https://lwn.net/Articles/284141/>), but is not seen as a feature that must be in the first release. Not copying the data for most drivers is a bigger win, but compression, especially for large firmware images might help. In those cases, though, both the compressed and uncompressed data will be in memory while the driver is downloading it. 

Getting this work included into 2.6.26 has been discussed, even though the merge window has closed. Woodhouse [thinks](<https://lwn.net/Articles/284150/>) it _might_ be possible: 

Well, it's supposedly too late, but it's dead simple and shouldn't have much chance of breaking anything, so I suppose as long as we don't include the korg1212 patch and the rest of the similar patches which we're still working on, that's not such an insane request. 

This is a fairly simple patch that adds some very useful functionality, especially for the embedded community. Woodhouse has recently [stepped up as one the kernel embedded maintainers](<http://lwn.net/Articles/283749/>), so we may see more things like this from him in the future. It is unlikely that Linus Torvalds will merge this feature so late in the 2.6.26 cycle, but inclusion into 2.6.27 seems quite probable.
