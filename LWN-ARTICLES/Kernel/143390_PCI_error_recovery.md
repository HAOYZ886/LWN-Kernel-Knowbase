---
title: PCI error recovery
url: https://lwn.net/Articles/143390/
date: "July 12, 2005"
category: "Device drivers-Support APIs; PCI"
author: 
---

The PCI bus is the interconnect of choice for the bulk of the architectures supported by Linux. Most peripherals on such systems - including disk, network, and USB controllers - communicate with the CPU via this bus. Linux device drivers (regardless of the bus used) must be written with the idea that the device being controlled can fail. Most drivers, however, assume that the bus used to communicate with the device will work flawlessly. This assumption exists because (1) it tends to be true, and (2) the Linux kernel has never provided an infrastructure which enables drivers to detect (and respond to) PCI errors. Work is under way to provide that infrastructure, however; there are currently two entirely different interfaces being proposed for this role. 

The first approach, [posted](<https://lwn.net/Articles/141975/>) by Linas Vepstas, works by way of callbacks. It enhances the `pci_driver` structure by adding a new set of methods: 

```
struct pci_error_handlers
    {
        enum pci_channel_state error_state;
        int (*error_detected)(struct pci_dev *dev, 
                              enum pci_channel_state error);
        int (*mmio_enabled)(struct pci_dev *dev);
        int (*link_reset)(struct pci_dev *dev);
        int (*slot_reset)(struct pci_dev *dev);
        void (*resume)(struct pci_dev *dev);
    };
```

A PCI driver is not required to supply any of these callbacks. Any driver which will perform PCI error recovery must provide at least `error_detected()`, however. That method will be called sometime after the PCI subsystem detects an error on the bus; the `error` parameter will be set to one of these values: 

```
enum pci_channel_state {
        pci_channel_io_normal = 0, /* I/O channel is in normal state */
        pci_channel_io_frozen = 1, /* I/O to channel is blocked */
        pci_channel_io_perm_failure, /* pci card is dead */
    };
```

The `error_detected()` method should shut down any ongoing I/O operations, but should not attempt to communicate with the adapter itself. This method can take locks and sleep; it is called from process context. The return value tells the error recovery subsystem how to proceed; it can be `PCIERR_RESULT_CAN_RECOVER` (the driver thinks it will be able to recover just by talking to the adapter), `PCIERR_RESULT_NEED_RESET` (a hard reset of the adapter will be required), or `PCIERR_RESULT_DISCONNECT` (the situation is hopeless, and the adapter should be considered permanently dead). 

If all drivers on an affected PCI segment think they can recover from the problem, the next step is to turn memory-mapped I/O back on and let the drivers try. To this end, each driver's `mmio_enabled()` callback will be invoked. This callback should do whatever port banging is required to get the adapter back into a reasonable state, then return one of `PCIERR_RESULT_RECOVERED` (it worked), `PCIERR_RESULT_NEED_RESET` (it failed, try resetting), or `PCIERR_RESULT_DISCONNECT` (it failed, abandon all hope). Regardless of the outcome, the driver should not restart I/O from this callback. 

The `link_reset()` method is similar to `mmio_enabled()`, but it is only applicable for PCI-Express adapters which might be fixable via a link reset operation. The return codes are the same as for `mmio_enabled()`. 

If a reset is called for, the PCI subsystem will perform the reset, then call `slot_reset()` to let the driver know. The driver should attempt to bring the adapter back to a working state, re-download firmware, etc., then return a status code indicating whether things worked or not. If reinitialization fails, it is possible that `slot_reset()` could be called more than once as the PCI subsystem employs an increasingly large hammer. 

Finally, if all seems to be well, the driver's `resume()` callback will be called; this is the point where I/O operations can be restarted. 

A very different approach is taken by [the IOCHK interface](<http://lwn.net/Articles/142767/>) posted by Hidetoshi Seto. This patch expects drivers to perform more of their own error checking, but gives more control over the timing of recovery operations. 

The IOCHK patch works by defining a new opaque type called `iocookie`. A driver which is about to engage in a conversation with one of its devices would initialize one of these cookies with: 

```
void iochk_clear(iocookie *cookie, struct pci_dev *dev);
```

The driver then performs its device operations, reading and writing memory-mapped I/O registers as necessary. At any point, the driver can check to see whether an error has occurred with: 

```
int iochk_read(iocookie *cookie);
```

A non-zero return indicates trouble; should that happen, the driver can respond by resetting the device, disconnecting it, or going into hysterics. There is no core support for operations like resetting adapters. 

The obvious question which has been raised is why two interfaces are needed. It seems that some situations are better handled by an asynchronous notification mechanism (such as implemented by Linas's patch), while others are better suited to a synchronous approach. So it may well be that, at some point in the future, the kernel will go from no PCI error handling interfaces to two of them. Before that happens, however, one assumes that some work will be done to unify the underlying support code and to make the two interfaces appear more like parts of a single API.
