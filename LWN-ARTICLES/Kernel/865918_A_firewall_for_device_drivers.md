---
title: A firewall for device drivers
url: https://lwn.net/Articles/865918/
date: "August 13, 2021"
category: "Device drivers; Security-Kernel hardening"
author: "By Jonathan Corbet August 13, 2021"
---

By **Jonathan Corbet**  
August 13, 2021

Device drivers, along with the hardware they control, have long been considered to be a trusted part of the system. This faith has been under assault for some time, though, and it fails entirely in some situations, including virtual machines that do not trust the host system they are running under. The recently covered [virtio-hardening work](<https://lwn.net/Articles/865216/>) is one response to this situation, but that only addresses a small portion of the drivers built into a typical kernel. What is to be done about the rest? The [driver-filter patch](<https://lwn.net/ml/linux-kernel/20210804174322.2898409-1-sathyanarayanan.kuppuswamy@linux.intel.com/>) from Kuppuswamy Sathyanarayanan demonstrates one possible approach: disable them altogether. 

Virtual machines typically have direct access to little or no physical hardware; instead, they interact with the world by way of emulated devices provided by the host. That puts the host in a position of power, since it is in total control over how those virtual devices work. If a driver has not been written with the idea that the devices it manages could be hostile, chances are good that said driver can be exploited to compromise the guest and exfiltrate data — even when the guest is running with encrypted memory that is normally inaccessible to the host. 

The virtio work hardens a handful of virtio drivers to prevent them from misbehaving if the host decides to not play by the rules. Getting there was a lot of work (which still has not reached the point of being merged), and there is a decidedly non-zero chance that vulnerabilities remain. Even if the virtio work is perfect, though, the kernel contains thousands of other drivers, most of which have not received anything close to the same amount of attention; few of them can be expected to be sufficiently robust to stand up to a malicious device. If the host can convince a guest to load the driver for such a device, the security game may well be over. 

One possible solution to this problem is to methodically go through and harden all those thousands of until-now neglected drivers. The result would surely be a better kernel, but holding one's breath for this outcome would be ill-advised. Even if the developer effort for such a project can be found, there is a lot of code that would have to be tested with a large array of devices, a significant number of which stopped being widely available many years ago. Any realistic plan must accept that many drivers will never be hardened in this way. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The alternative is to simply make those drivers unavailable; a driver that cannot run at all is unlikely to compromise the system. Most virtual machines only need a handful of drivers; the rest are just dangerous dead weight. The obvious thing to do is to build a kernel with only the needed drivers, yielding a result that is not only safer, it will also be much smaller. The problem with this idea is that distributors hate the idea of shipping multiple kernels with different configurations. Each one adds to the build, test, and support loads, and it only takes a few configuration options to create a large array of kernel images. Distributors are thus highly motivated to ship a single kernel image if possible. 

This is where Sathyanarayanan's patch set comes in; it provides a way for the system administrator to control which drivers are allowed to run. It adds two new command-line options — `filter_allow_drivers` and `filter_deny_drivers` — for that purpose; specific drivers can be added to either list using a "`bus:driver`" notation. The string "`ALL`" matches anything. So, for example, booting a system with: 

```
filter_allow_drivers=ALL:ALL
```

will allow all drivers to run — the default situation. The allow list is applied first and overrides the deny list, so a configuration like this: 

```
filter_allow_drivers=pci:e1000 filter_deny_drivers=ALL:ALL
```

will allow the e1000 network adapter driver to run, but will block everything else. There is also a new driver attribute in sysfs (called `allowed`) that can be used to change a driver's status at run time. 

Driver subsystem maintainer Greg Kroah-Hartman [was not impressed](<https://lwn.net/ml/linux-kernel/YQrqhYEL64CSLRTy@kroah.com/>) with this submission; he suggested either building a special kernel image or using the existing mechanisms to block unwanted device drivers instead. These could include denying them in the system's `modprobe` configuration or using the knobs in sysfs to unbind drivers from their devices. As Andi Kleen [explained](<https://lwn.net/ml/linux-kernel/f2b1d564-8174-f8e9-9fee-12e938c6d846@linux.intel.com/>), though, these mechanisms do not quite satisfy the requirements. Configuring `modprobe` does not help with built-in drivers and, in any case, the intent is to prevent untrusted drivers from running at all. By the time user space can manually unbind a driver, it has already set itself up in the kernel and may already be trying to drive a malicious device. 

Another way of looking at the situation, Kleen [added](<https://lwn.net/ml/linux-kernel/e96bc698-89ca-3c67-d0ba-adecc50e8a53@linux.intel.com/>), is to see a guest running on a potentially hostile host as being like a server on the Internet. The server almost certainly runs a firewall to restrict access to ports that are known (or at least hoped) to be safe; the driver filter is the equivalent of the firewall for the guest. That simplifies the hardening problem to the point that it might be feasible. 

Whether these arguments will convince Kroah-Hartman remains to be seen; the conversation went quiet without reaching any sort of definitive conclusion. The problem that is driving this work seems real, though; if the current solution does not make the cut, we are likely to see other attempts to do something similar in the future. Devices have gone from hiding behind the kernel to being a part of the kernel's attack surface; security-focused developers will naturally want to reduce that surface as much as possible.
