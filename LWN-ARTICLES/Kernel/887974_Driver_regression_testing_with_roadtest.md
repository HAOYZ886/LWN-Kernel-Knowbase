---
title: Driver regression testing with roadtest
url: https://lwn.net/Articles/887974/
date: "March 18, 2022"
category: "Development tools-Testing"
author: "By Jonathan Corbet March 18, 2022"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 18, 2022

The kernel community has a number of excuses for the relative paucity of regression-test coverage in the project, some of which hold more water than others. One of the more convincing reasons is that a great deal of kernel code is hardware-specific, and nobody can ever hope to put together a testing system with even a small fraction of all the hardware that the kernel supports. A new driver-testing framework called [roadtest](<https://lwn.net/ml/linux-kernel/20220311162445.346685-1-vincent.whitchurch@axis.com/>), posted by Vincent Whitchurch, may make that excuse harder to sustain, though, at least for certain kinds of hardware. 

One of the problems with hardware is its sheer variety. Consider a device as conceptually simple as a GPIO port which, at its core, drives a single electrical line to either a logical true or false value. GPIO drivers should be simple things, and many of them are, but vendors like to add their own flourishes with each new release. As a result, there are [well over 150 GPIO drivers](<https://elixir.bootlin.com/linux/latest/source/drivers/gpio>) in the kernel source, many of which can drive more than one variant of a device. There is no way to build a system with all of those devices in it; most of them are part of a larger peripheral or system-on-chip, and many of them have not been commercially available for years. 

Of course, each of those drivers was, at one point, tested on the hardware it drives. They would normally be expected to continue to work. But the kernel is constantly changing, and changes often affect drivers as well. Developers making those changes do their best to avoid breaking anything, but they have no way to test changes that affect most drivers; even subsystem maintainers will normally only have a subset of the devices available for testing. So there is always a possibility that regressions will slip in and go unnoticed until somebody's device stops working. 

Roadtest aims to circumvent this problem by eliminating the need to actually have the hardware present to test whether a driver still works. This is done by pairing driver tests with mock devices that can run anywhere; when a developer makes a set of regression tests for a specific driver, that work includes the mocked version of the target device(s) as well. The tests are then run under a specially built [User-Mode Linux](<http://user-mode-linux.sourceforge.net/>) kernel, with the mocked hardware filling in for the real thing. 

Forcing a test author to also implement an emulated version of the device under test sounds like a high bar to overcome. The good news is that the mocked devices need not encapsulate the full complexity of the real thing; they simply need to respond well enough to verify that the driver is behaving in the expected way. Emulating the device's programming interface (without actually doing the things a real device would do) may well be sufficient. 

Consider, for example, [this test](<https://lwn.net/ml/linux-kernel/20220311162445.346685-8-vincent.whitchurch@axis.com/>) from the patch set, which verifies the driver for the [opt3001 light-sensor driver](<https://elixir.bootlin.com/linux/latest/source/drivers/iio/light/opt3001.c>). Both the tests and the mocked devices are written in Python; the core part of the implementation for the mocked opt3001 device looks like this: 

```
class OPT3001(SMBusModel):
            def __init__(self, **kwargs: Any) -> None:
                super().__init__(regbytes=2, byteorder="big", **kwargs)
                # Reset values from datasheet
                self.regs = {
                    REG_RESULT: 0x0000,
                    REG_CONFIGURATION: 0xC810,
                    REG_LOW_LIMIT: 0xC000,
                    REG_HIGH_LIMIT: 0xBFFF,
                    REG_MANUFACTURER_ID: 0x5449,
                    REG_DEVICE_ID: 0x3001,
                }
        
            def reg_read(self, addr: int) -> int:
                val = self.regs[addr]
        
                if addr == REG_CONFIGURATION:
                    # Always indicate that the conversion is ready.  This is good
                    # enough for our current purposes.
                    val |= REG_CONFIGURATION_CRF
        
                return val
        
            def reg_write(self, addr: int, val: int) -> None:
                assert addr in self.regs
                self.regs[addr] = val
```

The opt3001 is an [SMBus](<https://en.wikipedia.org/wiki/System_Management_Bus>) device, programmable via writes to (and reads from) a set of registers. Using the SMBus emulation provided with roadtest, this mock device simply implements a handful of registers. It is hard to imagine a simpler implementation; the read side doesn't even bother to check whether a requested register number is valid, presumably on the assumption that the crash resulting from a bad read request would say "test failure" with adequate volume. 

The roadtest framework will take the mock device implementation and connect it to the driver (in the User-mode Linux instance) as if it were a real device. The test itself runs as a user-space process in that instance; it tweaks some of those registers to simulate the arrival of data, then reads that data using the [IIO](<https://www.kernel.org/doc/html/v4.12/driver-api/iio/index.html>) API: 

```
def test_illuminance(self) -> None:
            data = [
                # Some values from datasheet, and 0
                (0b_0000_0000_0000_0000, 0),
                (0b_0000_0000_0000_0001, 0.01),
                (0b_0011_0100_0101_0110, 88.80),
                (0b_0111_1000_1001_1010, 2818.56),
            ]
            with self.driver.bind(self.dts["addr"]) as dev:
                luxfile = dev.path / "iio:device0/in_illuminance_input"
    
                for regval, lux in data:
                    self.hw.reg_write(REG_RESULT, regval)
                    self.assertEqual(read_float(luxfile), lux)
```

The register writes (the `self.hw.reg_write()` call above) go straight to the mock device. The reads, instead, are directed to the driver, which will interact with the mock device to obtain the requested data. If the driver is working properly, it will read the simulated data from the mock device and return the results that the test is expecting. 

This is a simple test; more complex tests could verify that the driver is setting up the hardware correctly, dealing with error conditions, and so on. Even so, there would appear to be limits to a mechanism like this; it will be difficult to use it to verify that, say, a Video4Linux driver is correctly managing the buffer queue when user-mapped buffers are in use with a planar YUV color scheme. But for simpler devices, of which there are many, a system like roadtest may provide a level of assurance that kernel developers currently do not have. 

A lot more information on roadtest can be found in [this documentation patch](<https://lwn.net/ml/linux-kernel/20220311162445.346685-7-vincent.whitchurch@axis.com/>), which includes a tutorial on adding a test for a new device. The patch set as a whole contains tests for a few device types; presumably that list would grow considerably if this framework were to be merged into the mainline. 

There have not been a lot of comments on the system so far, so it is hard to be sure about what roadtest's prospects for merging are. Brendan Higgins was [clear enough](<https://lwn.net/ml/linux-kernel/CAFd5g47O2PbqaUZRoioRROtywTm=6t7cVgHqO7qc0ZGewQk16A@mail.gmail.com/>) on his opinion of roadtest, though: ""I love the framework; this looks very easy to use"". Testing frameworks like roadtest should not bother anybody who does not choose to use them and, if they are made comprehensive enough, they can significantly increase the chances of catching regressions before they get into a released kernel. So it is hard to see a reason why roadtest wouldn't eventually become part of the mainline kernel — unless, of course, kernel developers would really rather not lose an excuse justifying the lack of regression testing for drivers.
