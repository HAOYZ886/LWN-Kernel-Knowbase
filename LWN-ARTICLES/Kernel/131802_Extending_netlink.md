---
title: Extending netlink
url: https://lwn.net/Articles/131802/
date: "April 12, 2005"
category: "Events reporting; Networking-Netlink"
author: 
---

The netlink mechanism implements a special sort of datagram socket for communication between the kernel and user space. Most of the users of netlink are currently in the networking subsystem itself - netlink protocols exist, for example, for the management of routing table entries and firewall rules. Netlink is also used by SELinux and the kernel event notification mechanism. 

Use of netlink is relatively straightforward - for kernel developers who have some familiarity with the networking subsystem. To be able to communicate via netlink, a kernel subsystem must first create an in-kernel socket: 

```
struct sock *netlink_kernel_create(int unit, 
                             void (*input)(struct sock *sk, int len));
```

Here, `unit` is the netlink protocol number (as defined in `<linux/netlink.h>`), and `input()` is a function to be called when data arrives on the given socket. The naming of `unit` dates back to an early netlink implementation, which worked with virtual devices; `unit` was the minor number of the relevant device. The `input()` callback can be `NULL`, in which case user space will not be able to write to the socket. 

If there is an `input()` callback, it will be called whenever data arrives. That data will be represented in one or more `sk_buff` structures (SKBs) queued to the socket itself. So the core of a typical `input()` function will look something like: 

```
struct sk_buff *skb;
    
        while ((skb = skb_dequeue(sk->sk_receive_queue)) != NULL) {
            deal_with_incoming_data(skb);
    	kfree_skb(skb);
        }
```

Sending data to user space involves allocating an SKB, filling it with the data, and writing it to the netlink socket. Here is how the kernel events mechanism does it: 

```
static int send_uevent(const char *signal, const char *obj,
    		           char **envp, int gfp_mask)
        {
    	struct sk_buff *skb;
    	char *pos;
    	int len;
    
    	len = strlen(signal) + 1;
    	len += strlen(obj) + 1;
    
    	/* allocate buffer with the maximum possible message size */
    	skb = alloc_skb(len + BUFFER_SIZE, gfp_mask);
    	pos = skb_put(skb, len);
    	sprintf(pos, "%s@%s", signal, obj);
    
    	/* copy the environment key by key to our continuous buffer */
    	if (envp) {
    	    int i;
    
    	    for (i = 2; envp[i]; i++) {
    		len = strlen(envp[i]) + 1;
    		pos = skb_put(skb, len);
    		strcpy(pos, envp[i]);
    	    }
    	}
    	return netlink_broadcast(uevent_sock, skb, 0, 1, gfp_mask);
        }
```

(Some error handling has been removed for brevity; see `lib/kernel_uevent.c` for the full version). The call to `netlink_broadcast()` sends the data in the SKB to every user-space process which is currently connected to the netlink socket. There is also `netlink_unicast()`, which takes a process ID and sends only to that process. Netlink writes can be restricted to specific "groups," allowing user-space processes to sign up for an interesting subset of the data written to a given socket. 

There is more to the netlink interface than has been presented here; see `<linux/netlink.h>` for the rest. 

Evgeniy Polyakov thinks that the netlink protocol is too complicated; it should not be necessary to understand the networking layer just to communicate with user space. His response is [connector](<https://lwn.net/Articles/131566/>), a layer on top of netlink which is designed to make things simpler. 

The connector code multiplexes all possible message types over a single netlink socket number. Individual messages are distinguished by way of a `cb_id` structure: 

```
struct cb_id
        {
    	__u32 idx;
    	__u32 val;
        };
```

`idx` can be thought of as a protocol type, and `val` as a message type within the given protocol. A kernel subsystem which is prepared to receive messages of a given type set up a callback with: 

```
int cn_add_callback(struct cb_id *id, char *name,
                            void (*callback)(void *msg));
```

That callback will be invoked every time a message with the given `id` is received from user space. The `msg` parameter to the callback function, despite its `void *` type, is always a pointer to a structure of this type: 

```
struct cn_msg
        {
    	struct cb_id 		id;
    	__u32			len;	/* Length of the following data */
    	__u8			data[0];
            /* Some fields omitted */
        };
```

The callback can process the given message data and return. 

Writing to a socket via connector is done with: 

```
void cn_netlink_send(struct cn_msg *msg, u32 __groups, int gfp_mask);
```

The `msg` contains the `cb_id` structure describing the message; `__groups` can be used to restrict the list of recipients, and `gfp_mask` controls how memory allocation is done. This call can fail (netlink is an unreliable service), but it returns no indication of whether it succeeded or not. 

For kernel code which needs to send significant amounts of data to user space, perhaps from hot paths, there is also a "CBUS" layer over the connector. That layer exports one function: 

```
int cbus_insert(struct cn_msg *msg, int gfp_flags);
```

This function does not send the message immediately; it simply adds it to a per-CPU queue. A separate worker thread will eventually come along, find the message, and send it on to user space. 

The code seems to work, though some concerns have been raised about the implementation. Not everybody feels that the connector solution is necessary, however. The core netlink API is not all that hard to use, so it is not clear that another layer needs to be wrapped around it. Those who do think that netlink could be made easier do not agree on how it should be done; some developers would like to see the netlink API itself changed rather than having another layer put on top of it. Various user-space needs (auditing, accounting, desktop functionality, etc.) are all creating pressure for more communication channels with the kernel. Some way of making that communication easier on the kernel side may well get added, eventually, but it is far from clear what form that code will take.
