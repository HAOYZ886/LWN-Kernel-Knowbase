---
title: IR decoding with BPF
url: https://lwn.net/Articles/759188/
date: "July 9, 2018"
category: "BPF-Device drivers; Device drivers-Support APIs"
author: "July 9, 2018 This article was contributed by Sean Young"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

July 9, 2018

This article was contributed by Sean Young

In the 4.18 kernel, a new feature was merged to allow infrared (IR) decoding to be done using [BPF](<https://lwn.net/Articles/740157/>). Infrared remotes use many different encodings; if a decoder were to be written for each, we would end up with hundreds of decoders in the kernel. So, currently, the kernel only supports the most widely used protocols. Alternatively, the `lirc` daemon can be run to decode IR. Decoding IR can usually be expressed in a few lines of code, so a more lightweight solution without many kernel-to-userspace context switches would be preferable. This article will explain how IR messages are encoded, the structure of a BPF program, and how a BPF program can maintain state between invocations. It concludes with a look at the steps that are taken to end up with a button event, such as a volume-up key event. 

Infrared remote controls emit IR light using a simple LED. The LED is turned on and off for shorter or longer periods, which is interpreted somewhat akin to morse code. When infrared light has been detected for a period, the result is called a "pulse". The time between pulses when no infrared light is detected is called a "space". 

Whenever a pulse or space is detected by an IR receiver, a BPF program will be executed (if one is attached). This program consists of a single function entry point that takes a pointer to a context. For IR decoders, this context is an `unsigned int` value. For a packet filter, the context would instead be a packet. In our case, the lower 24 bits of the `int` value contain the duration of the pulse or space, in microseconds. The top eight bits define the type of the event, which can either be `LIRC_MODE2_PULSE`, `LIRC_MODE2_SPACE`, or `LIRC_MODE2_TIMEOUT`. The return value of the BPF program is ignored. 

If a space between two pulses gets excessively long, it could delay the decoding of a button press. For example, we might want to know that the IR message has really ended by measuring the space after the last pulse has occurred. Since a space is a time between two pulses, we would have to wait for the next pulse from the next IR message to occur before we would get this value. So, for this reason, there is a timeout. If a space lasts longer than the timeout, it is reported as `LIRC_MODE2_TIMEOUT`. This is typically set at 125ms. 

A BPF program can be written in a number of different ways, but the easiest way is to use `clang` with the target BPF. This allows the BPF program to be written in a sort of restricted C that does not allow the use of C-library functions or loops, for example. 

To create an IR decoder in BPF, we start with: 

```
static int eq_margin(int duration, int expected, int margin)
        {
    	return (duration >= (expected - margin)) 
         	    && (duration <= (expected + margin));
        }
    
        int bpf_decoder(unsigned int *sample)
        {
    	int duration = *sample & LIRC_VALUE_MASK;
    	bool pulse = (*sample & LIRC_MODE2_MASK) == LIRC_MODE2_PULSE;
    
    	if (pulse && eq_margin(duration, 300, 100) {
    		// seen short pulse of about 300 microseconds
    	}
        }
```

Typically, IR receivers have a precision of 50µs at most. I would recommend checking for durations of at least 100µs around the value you expect. 

Now we can parse a single pulse or space, but every IR message consists of several pulses and spaces in quick succession. In a regular C program, we would use a static variable, a global variable, or some heap memory to maintain our state while waiting for the next event. Unfortunately none of those options are available in BPF. Instead, we use BPF maps, which are a generic key-value store where the key is always an `unsigned int` and the value is a generic blob; we can store whatever we want. This is how we declare a BPF map to hold the IR-decoding state: 

```
struct decoder_state {
    	unsigned int bits;
    	unsigned int count;	
        };
    
        struct bpf_map_def SEC("maps") decoder_state_map = {
    	.type = BPF_MAP_TYPE_ARRAY,
    	.key_size = sizeof(unsigned int),
    	.value_size = sizeof(struct decoder_state),
    	.max_entries = 1,
        };
```

There are a few different types of BPF maps, the main ones being "array" and "hash". Since we are only looking to store one structure, an array is more than sufficient; we thus specify `max_entries` as one. The `key_size` has to be the size of an `unsigned int`, no other key size is supported. The `value_size` is the size of our blob of data. We've declared a struct for this purpose, and we use `sizeof()` to ensure we have the right storage for it. 

There are a number functions available to use BPF maps from our BPF code. For example, to get an our entry in `decoder_state_map` BPF map, we can call: 

```
int key = 0;
        struct decoder_state *s = bpf_map_lookup_elem(&decoder_state_map, &key);
```

Unfortunately, if we try to use the pointer to the map, we will get an error when we load our BPF program: "R6 invalid mem access 'map_value_or_null'". This is the kernel's BPF verifier complaining; it checks to ensure that a BPF program does not do anything it should not, like try to access out-of-bounds memory. It also checks for other conditions, like relying on undefined behavior or loops. 

The problem here is that `bpf_map_lookup_elem()`, the function used to obtain a value from a BPF map, might return NULL if the key is beyond the last element. The elements of an array are pre-allocated, and we are looking for element zero out of a total of one, so this lookup should never fail. However, the BPF verifier is not aware of this so, in order to keep the verifier happy, we have to add: 

```
if (!s)
    	return 0;
```

The pointer we get from `bpf_map_lookup_elem()` is a direct pointer to the array, so we do not have to call `bpf_map_update_elem()` after making changes. The BPF verifier will check that we only use our pointer with the right offsets within our array entry; otherwise our program will not load. 

Now that we have memory to store state, we can implement decoding. When we have decoded the IR to a button event, we can submit that event to the input subsystem using the BPF function `bpf_rc_keydown()`. It takes four arguments, being the BPF context, the protocol, the scancode, and the toggle bit: 

  * The context for BPF is the pointer that was passed to the main BPF function; so we simply pass `sample` here. 
  * The IR protocol can be used by user space to determine which protocol produced any given scancode; at the moment, nothing uses it. 
  * The scancode is the value that was decoded. IR protocols generally encode some sort of value, and that value does not necessarily represent a key or a button. A particular remote might assign particular values with buttons; so, we need a mapping from scancode to key code. This is done using remote-control keymaps, which usually live in `/lib/udev/rc_keymaps/` if the `v4l-utils` package is installed (or the `ir-keytable` package on Ubuntu or Debian). 
  * Some IR protocols include a toggle bit. Since the IR message is repeated every 90ms or so, it is impossible to distinguish a key being held from a key released and pressed again (toggled). In the latter case, the toggle bit will change value, so `rc-core` knows to generate both key-up and key-down events. 

So those are the four arguments to `bpf_rc_keydown()`. Now, we can show a complete example of a fictional IR decoder. 

```
#include <linux/lirc.h>
        #include <linux/bpf.h>
    
        #include "bpf_helpers.h"
    
        enum state {
    	STATE_INACTIVE,
    	STATE_FIRST_PULSE,
    	STATE_SECOND_PULSE
        };
    
        struct decoder_state {
           enum state state;
           unsigned int space;
        };
    
        struct bpf_map_def SEC("maps") decoder_state_map = {
    	.type = BPF_MAP_TYPE_ARRAY,
    	.key_size = sizeof(unsigned int),
    	.value_size = sizeof(struct decoder_state),
    	.max_entries = 1,
        };
    
        SEC("fictional_ir")
        int decode(unsigned int *sample)
        {
    	int key = 0;
    	struct decoder_state *s = bpf_map_lookup_elem(&decoder_state_map, &key);
    
    	if (!s)
    	    return 0;
    
    	int duration = LIRC_VALUE(*sample);
    
    	switch (s->state) {
    	case STATE_INACTIVE:
    	    if (LIRC_IS_PULSE(*sample) && duration == 500) {
    		s->state = STATE_FIRST_PULSE;
    	    }
    	    break;
    	case STATE_FIRST_PULSE:
    	    if (LIRC_IS_SPACE(*sample)) {
    		s->space = duration;
    		s->state = STATE_SECOND_PULSE;
    	    } else {
    		s->state = STATE_INACTIVE;
    	    }
    	    break;
    	case STATE_SECOND_PULSE:
    	    if (LIRC_IS_PULSE(*sample) && duration == 500) {
    	 	bpf_rc_keydown(sample, 64, s->space / 100, 0);
    	    }
    	    s->state = STATE_INACTIVE;
    	    break;
    	}
    
    	return 0;
        }
    
        char _license[] SEC("license") = "GPL";
```

Several operations are multiplexed through the [`bpf()` system call](<http://man7.org/linux/man-pages/man2/bpf.2.html>) for managing BPF programs and BPF maps, and for attaching them to devices. To create a BPF program, the `BPF_PROG_LOAD` is used. We have to provide a pointer to the BPF instructions, the instruction count, and a program name. If the system call is successful, we will get a file descriptor. 

We can create BPF maps with the `BPF_MAP_CREATE` command, which also returns a file descriptor on success. Once we have the program and maps created, we can attach the program to a LIRC device (e.g. `/dev/lirc0`) using the `BPF_PROG_ATTACH` command. We have to provide a file descriptor for the LIRC device and the BPF program file descriptor. Once the file descriptor is attached, we can safely exit our process and the BPF program won't be freed when its file descriptor is closed. 

Currently there is a hard-coded limit of 64 BPF programs that may be attached to one LIRC device. Any more, and `BPF_PROG_ATTACH` will return `E2BIG`. Every time a new pulse or space occurs, all the BPF programs will be executed. This makes it possible to load multiple BPF decoders, so that different remotes can be used at the same time. 

As you might expect there are also commands for querying and detaching BPF programs. 

The BPF example above can be compiled it with: 

```
clang --target=bpf -O2 -c foobar.c
```

You'll need to compile it with kernel headers from 4.18 (or later), and the `bpf_helpers.h` from the same tree. This produces `foobar.o`, an ELF object file. 

Using `ir-keytable`, you can load this BPF program. You'll need the [BPF patches](<https://git.linuxtv.org/syoung/v4l-utils.git/log/?h=bpf>), which have not been merged yet at the time of writing. In order to simulate this, the `rc-loopback` pseudo-receiver can be used, so no IR hardware is needed. Here are the steps to make this work: 

```
modprobe rc-loopback
        ir-keytable -p ./foobar.o
```

In order to test this setup, create a file `test` with the following contents: 

```
pulse 500
        space 1500
        pulse 500
```

Now, run: 

```
ir-keytable -k 15:KEY_VOLUMEUP -t
```

in one terminal, and: 

```
ir-ctl -s test
```

in another. You should get this output: 

```
855.168999: lirc protocol(64): scancode = 0xf
        855.169009: event type EV_MSC(0x04): scancode = 0x0f
        855.169009: event type EV_KEY(0x01): key_down: KEY_VOLUMEUP
        855.169009: event type EV_SYN(0x00).
```

The `ir-keytable` patches above also include a Python script that converts `lircd` remote configuration so that it can be used with `ir-keytable`. This should make it possible to do without the `lirc` daemon. However, some protocol decoders require very basic loops, which currently the BPF verifier does not allow at all. 

Even with all `lircd` remote configurations supported, that would still not cover all possible remote controls. A possible solution can be found in [IRP notation](<http://www.hifi-remote.com/wiki/index.php?title=IRP_Notation>), a general form of description for IR protocols; it would be nice to generate BPF from that, and have support for a very broad array of remotes, without having to open-code each one. Lastly, other things than button presses are encoded in IR, for example target temperatures in air conditioning remote controls, or some remote controls include a directional pad. Supporting such devices with BPF decoders will require some further work.
