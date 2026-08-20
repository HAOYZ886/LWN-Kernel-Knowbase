---
title: "Writing a WMI driver - an introduction"
url: https://lwn.net/Articles/391230/
date: "June 8, 2010"
category: "Device drivers-Support APIs; Windows Management Instrumentation WMI"
author: "June 8, 2010 This article was contributed by Corentin Chary"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

June 8, 2010

This article was contributed by Corentin Chary

[Windows Management Instrumentation (WMI)](<http://en.wikipedia.org/wiki/Windows_Management_Instrumentation>) is a set of extensions to the Windows Driver Model that provides an operating system interface for dealing with platform devices. WMI objects can be embedded within ACPI, a configuration which Microsoft recommends. Like ACPI, WMI is not really standardized and vendors still implement their own custom interfaces. In this article, I will, through the creation of a simple WMI driver, discuss the process of discovering WMI interfaces and working with them. 

As WMI is embedded into ACPI tables, you should really start with [Matthew's article on ACPI drivers](<http://lwn.net/Articles/367630/>) before reading this one. You'll need to know how to extract, decompile and read your DSDT before going further. The DSDT (Discrete System Descriptor Table) lives in one of the ACPI tables provided to the operating system by the BIOS; it contains configuration information and executable code. On Linux, it can be found in `/sys/firmware/acpi/DSDT`; you will need to decompile it with `iasl` using the `-d` option. `iasl` is the Intel ACPI compiler; it is probably already packaged in your favorite distribution, but if it's not you can always grab the source from [acpica.org](<http://www.acpica.org/downloads/>). 

In this article we'll focus on the history of the eeepc-wmi driver as an example; the DSDT used for this article (Eeepc 1201nl) can be downloaded [here](<https://lwn.net/Articles/391249/>). The interesting part of the DSDT for making an ACPI or WMI driver is the ACPI device descriptions. They are defined with the `Device (XXXX)` keyword, where `XXXX` is the four-character name of the device. ACPI devices are also identified by an HID string using the same namespace as ISA PNP devices. This is why, most of the time, standardized HID names start with `PNP`. For WMI Devices, this HID will always be `PNP0C14` (or `pnp0c14`). 

The first Eeepc systems were shipped with an ACPI device called `ASUS010`; Linux had an ACPI driver for that device called `eeepc-laptop`. Then, ASUS started shipping a BIOS with "Windows 7 support", and `eeepc-laptop` didn't want to load any more, because those BIOSes were disabling the `ASUS010` device when Windows 7 was detected, and Linux has been identifying itself as Windows 7 since 2.6.32. No `eeepc-laptop` driver means: no hotkeys, no rfkill, no LEDs, and sometimes even no backlight, because on some models you need to boot with `acpi_backlight=vendor` if you want a working backlight. 

A quick workaround was to boot with `acpi_osi="!Windows 2009"` or `acpi_osi=Linux`. But there's a better way: those BIOS updates also added a new ACPI device. It's easy to notice that this is a WMI device, thanks to the reserved _HID `PNP0C14` and the explicit `ASUSWMI` UID. From the DSDT: 

```
Device (AMW0)
        {
            Name (_HID, EisaId ("PNP0C14"))
        	Name (_UID, "ASUSWMI")
        	...
        }
```

So we have a WMI device, and we need to find what we can do with it. The first thing to do is to dump the GUID mapping of the WMI device. A good way to do it is to use [wmidump](<http://xf.iksaif.net/dev/wmidump.html>), it will parse the buffer returned by the _WDG method, and display it in a humanly readable form. The `_WDG` method is defined in the WMI device and provides mapping for data blocks, events, and WMI methods. The result of _WDG evaluation is a buffer containing an array of structures, each entry describing a GUID.

Here is the output of wmidump for Eeepc 1201nl:

```
97845ED0-4E6D-11DE-8A39-0800200C9A66:
            object_id: BC
            notify_id: 42
            reserved: 43
            instance_count: 1
            flags: 0x2 ACPI_WMI_METHOD 
        466747A0-70EC-11DE-8A39-0800200C9A66:
            object_id: BD
            notify_id: 42
            reserved: 44
            instance_count: 1
            flags: 0x2 ACPI_WMI_METHOD 
        ABBC0F72-8EA1-11D1-00A0-C90629100000:
            object_id: ?
            notify_id: D2
            reserved: 00
            instance_count: 1
            flags: 0x8 ACPI_WMI_EVENT 
        05901221-D566-11D1-B2F0-00A0C9062910:
            object_id: MO
            notify_id: 4D
            reserved: 4F
            instance_count: 1
            flags: 0
```

We can see four different GUIDs. The first two are flagged with `ACPI_WMI_METHOD`, while the third is flagged with `ACPI_WMI_EVENT`. `ACPI_WMI_METHOD` means that, in the same ACPI device, there is a `WMXX` method, where `XX` is the `object_id` of this GUID. Thus, we will find a method called `WMBC` for GUID `97845ED0-4E6D-11DE-8A39-0800200C9A66`, and `WMBD` for `466747A0-70EC-11DE-8A39-0800200C9A66`. `ACPI_WMI_EVENT` is used to describe a GUID that will send events; hotkeys for example are reported using WMI events on Eeepc systems.

WMI support in Linux is provided by the `wmi` driver (`CONFIG_ACPI_WMI`) and linux/acpi.h. Using this framework, we can write a basic WMI driver that will load only if a given GUID is available. For that, we will use `wmi_has_guid(const char *guid);`. That function is easy to use: pass the GUID and it will return a true value if this GUID can be found. For this example we will use the `ABBC0F72-8EA1-11D1-00A0-C90629100000`. Here is a typical initialization function for a WMI driver:

```
#define EEEPC_WMI_EVENT_GUID	"ABBC0F72-8EA1-11D1-00A0-C90629100000"
    
        static int __init eeepc_wmi_init(void)
        {
    	if (!wmi_has_guid(EEEPC_WMI_EVENT_GUID)) {
    	    pr_warning("No known WMI GUID found\n");
    	    return -ENODEV;
    	}
    	return 0;
        }
```

Cool! A driver which does nothing :)

#### Events

Now, we want to catch hotkey events and send real input events when a hotkey is pressed. This requirement is common in platform drivers like eeepc-wmi and eeepc-laptop, so Dmitry Torokhov wrote the sparse keymap library to ease the implementation of such drivers. The `sparse-keymap` module (`CONFIG_INPUT_SPARSEKMAP`) allows the programmer to associate input events with custom codes (integers) and provides helpers to search a for code in a given keymap and report the resulting event through an input device. 

Input events that you'll send to your input device are defined in `<linux/input.h>`. Key events are prefixed with `KEY_`, for example "`a`" is `KEY_A`, F11 is `KEY_F11`, and the key used to toggle a wireless Lan device is `KEY_WLAN`. There are more than 380 distinct keys, so you should be able to find one that suits your needs. 

Defining a sparse keymap is simple:

```
#include <input/sparse-keymap.h>
    
        static const struct key_entry eeepc_wmi_keymap[] = {
    	{ KE_KEY, 0x42, { KEY_F13 } },
            { KE_END, 0},
        };
```

Then all you need to do is to initialize an input device, bind it with your sparse keymap, and call `sparse_keymap_report_event()` when you receive an event. I'll not describe the whole sparse-keymap API here (maybe in another article, who knows?), but if you want to see a (clean) real world example, please read `eeepc-wmi.c`.

Let's go back to our main topic: how can we receive WMI events? `wmidump` told us that one of the GUIDs was flagged ACPI_WMI_EVENT; this means that it is able to send events. To catch these events, we have to install a notify handler on this GUID with:

```
typedef void (*wmi_notify_handler) (u32 value, void *context);
        acpi_status wmi_install_notify_handler(const char *guid, 
    					   wmi_notify_handler handler,
    				       	   void *data);
```

The `void *data` argument passed to `wmi_install_notify_handler()` can be retrieved in `void *context` when the handler is called, and can be used to store context information. The important thing here is `value`: you can pass this value to `wmi_get_event_data()`, which fills an `acpi_buffer` that can be cast into an `acpi_object`. And most of the time for hotkeys, this object is an integer. Don't forget to call `wmi_install_notify_handler()` _after_ input and keymap initialization, because the handler is likely to use the input to device, so it has to be initialized. 

Here is how to register (and unregister) the WMI handler. In this example, sparse_keymap and input device handling have been removed for clarity purposes. 

```
static int __init eeepc_wmi_init(void)
        {
    	...
    	err = eeepc_wmi_input_setup(); // Setup sparse_keymap and input device
    	if (err)
    	    return err;
    
    	status = wmi_install_notify_handler(EEEPC_WMI_EVENT_GUID,
    					eeepc_wmi_notify, NULL);
    	if (ACPI_FAILURE(status)) {
    	    ... // Free sparse_keymap and input device
    	    return -ENODEV;
    	}
    	return 0;
        }
    
        static void __exit eeepc_wmi_exit(void)
        {
    	wmi_remove_notify_handler(EEEPC_WMI_EVENT_GUID);
    	... // Free sparse_keymap and input device
        }
```

Below you'll see the code for the handler. Here we don't need the `context` variable and we assume that `eeepc_wmi_input_dev` is accessible.

```
static void eeepc_wmi_notify(u32 value, void *context)
        {
    	struct acpi_buffer response = { ACPI_ALLOCATE_BUFFER, NULL };
    	union acpi_object *obj;
    	acpi_status status;
    	int code;
    
    	status = wmi_get_event_data(value, &response);
    	if (status != AE_OK) {
    	    pr_err("bad event status 0x%x\n", status);
    	    return;
    	}
    
    	obj = (union acpi_object *)response.pointer;
    
    	if (obj && obj->type == ACPI_TYPE_INTEGER) {
    		code = obj->integer.value;
    
    	    if (!sparse_keymap_report_event(eeepc_wmi_input_dev, code, 1, true))
    		pr_info("Unknown key %x pressed\n", code);
    	}
    
    	kfree(obj);
        }
```

Our keymap is empty for the moment, and because we are lazy, we don't want to read the whole DSDT to see what kinds of events are reported. An alternative is to implement a basic driver with an empty keymap, and make it dump every event. Then press some buttons, check dmesg, and fill the keymap! For example, pressing Fn+F2 will show "Unknown key 0x5d pressed." Fn+F2 is the wireless toggle key, so let's fill the keymap accordingly: 

```
static const struct key_entry eeepc_wmi_keymap[] = {
    	{ KE_KEY, 0x5d, { KEY_WLAN } },
            { KE_END, 0},
        };
```

#### Methods

Now, you should be able to create a basic driver for WMI event handling. But what about setting the brightness, enabling a GPS device or blinking an LED? If you go back to the `wmidump` output from the beginning, GUID `97845ED0-4E6D-11DE-8A39-0800200C9A66` has the `ACPI_WMI_METHOD` flag set, and its `object_id` is `BC`. That means that there is an ACPI `WMBC` method that can be called. This function has three parameters; the first is a ULONG that has the instance index being executed; the second contains the method ID for the method being executed; and the third is a buffer that contains the input for the method call.

To call such a method, the WMI module provides a function called `wmi_evaluate_method()`. It takes a GUID, an instance (we only have one here, see the output of `wmidump`), a method identifier and an input buffer. This buffer is used to pass custom parameters to the underlying method. It also takes an output buffer that will contain the return value of the method (if any).

```
acpi_status wmi_evaluate_method(const char *guid, u8 instance, u32 method_id,
                                        const struct acpi_buffer *in,
                                        struct acpi_buffer *out);
```

We will try to implement backlight control for this laptop, using WMI of course! Most of the time on x86 laptops, the backlight is handled by the generic ACPI video module. But sometimes, the generic ACPI backlight interface is broken, so you may want to use a vendor specific module to control the backlight. To do that, boot with `acpi_backlight=vendor`. We won't talk a lot about the backlight class, and we'll focus on the WMI specific part. But if you want to know more, read the complete eeepc-wmi driver. 

The first thing to do is to find how the backlight can be controlled. I won't describe the entire (painful) process of digging into the DSDT to find out how to control the backlight, and we will assume that the vendor gave you the WMI device documentation (and a pony!). But in a real world, you'll have to start from the `WMXX` method of your device (where `XX` is the `object_id` of your GUID) to find something related to what you want.

To control devices on an Eeepc, the WMI interface exposes two methods. The first one is `DEVS` which is used to set something in a device; its identifier is `0x53564544` and it takes two parameters: the device ID and the value you want to set. For the backlight, this device ID is `0x00050012` and the value is the brightness value (between zero and 15). This parameter can be translated into the following C structure; 

```
struct bios_args {
           u32     dev_id;
           u32     ctrl_param;
        };
```

The second method is named `DSTS`; it can be used to get the state of a device. Its identifier is `0x53544344` and it takes only one parameter: the device ID, which is the same used for `DEVS`

In summary: we have the GUID of our device, the ID of the methods we want to call and their custom magic parameters. Let's translate that to C and put it at the begining of our driver.

```
#define EEEPC_WMI_MGMT_GUID    "97845ED0-4E6D-11DE-8A39-0800200C9A66"
        #define EEEPC_WMI_METHODID_DEVS        0x53564544
        #define EEEPC_WMI_METHODID_DSTS        0x53544344
        #define EEEPC_WMI_DEVID_BACKLIGHT      0x00050012
```

The next thing to do is to write two helpers for `DEVS` and `DSTS` because they can be used not only for the backlight, but also probably to implement rfkill for Bluetooth and WIFI.

`DEVS` is used to set a state for a given device. It takes a device ID, and a custom parameter, they are passed using the `bios_args` structure in the `input` buffer. This helper is pretty simple.

```
static acpi_status eeepc_wmi_set_devstate(u32 dev_id, u32 ctrl_param)
        {
           struct bios_args args = {
               .dev_id = dev_id,
               .ctrl_param = ctrl_param,
           };
           struct acpi_buffer input = { (acpi_size)sizeof(args), &args };
    
           return wmi_evaluate_method(EEEPC_WMI_MGMT_GUID, 1, 
                                      EEEPC_WMI_METHODID_DEVS, &input, NULL);
        }
```

Calling `DSTS` is a little more complicated because it returns a value. In `wmi_evaluate_method()` we put the `dev_id` in `input`, and create an `output` buffer that will hold the return value. Then we check that the return value is really an integer (because we want an integer for brightness level, and we know that the DSDT should return one).

```
static acpi_status eeepc_wmi_get_devstate(u32 dev_id, u32 *ctrl_param)
        {
    	struct acpi_buffer input = { (acpi_size)sizeof(u32), &dev_id };
    	struct acpi_buffer output = { ACPI_ALLOCATE_BUFFER, NULL };
    	union acpi_object *obj;
           	acpi_status status;
           	u32 tmp = 0;
    
           	status = wmi_evaluate_method(EEEPC_WMI_MGMT_GUID, 1, 
           		 		     EEEPC_WMI_METHODID_DSTS, &input, &output);
           	if (ACPI_FAILURE(status))
                return status;
    
           	obj = (union acpi_object *)output.pointer;
           	if (obj && obj->type == ACPI_TYPE_INTEGER)
                tmp = (u32)obj->integer.value;
    	if (ctrl_param)
    	    *ctrl_param = tmp;
    
    	kfree(obj);
    	return status;
        }
```

Now, we have two helpers that can easily be used to set and get the state for a given device. We know the `dev_id` for the backlight, and we just need to link that with `backlight_device` callbacks using `0x00050012` as the dev_id. 

```
static int read_brightness(struct backlight_device *bd)
        {
           	u32 ctrl_param;
           	acpi_status status;
    
           	status = eeepc_wmi_get_devstate(EEEPC_WMI_DEVID_BACKLIGHT, &ctrl_param);
           	if (ACPI_FAILURE(status))
                return -1;
    	return ctrl_param & 0xFF;
        }
    
        static int update_bl_status(struct backlight_device *bd)
        {
    	u32 ctrl_param;
           	acpi_status status;
    
           	ctrl_param = bd->props.brightness;
    	status = eeepc_wmi_set_devstate(EEEPC_WMI_DEVID_BACKLIGHT, ctrl_param);
    	if (ACPI_FAILURE(status))
                return -1;
    	return 0;
        }
```

And we're done! Eeepc WMI device is a simple WMI device, but the principle should the same for others. I chose this one, because we waited a long time for this driver; Yong Wang finally wrote it for 2.6.35. This driver is young and really easy to read, so it is a good example.
