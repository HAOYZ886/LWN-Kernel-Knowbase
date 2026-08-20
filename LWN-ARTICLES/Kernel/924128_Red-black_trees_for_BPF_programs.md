---
title: "Red-black trees for BPF programs"
url: https://lwn.net/Articles/924128/
date: "February 27, 2023"
category: "BPF; Releases-6.3"
author: "By Jonathan Corbet February 27, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 27, 2023

Most of the kernel's code is written in C and intended to be run directly on the underlying hardware. That situation is changing in a few ways, though; one of those is the ability to write kernel code for the BPF virtual machine. The 6.3 kernel release will include a new API making the [red-black tree](<https://lwn.net/Articles/184495/>) data structure available to BPF programs. Beyond being an interesting feature in its own right, this new API shows how BPF is bringing a different approach to kernel programming — and to the C language in general. 

The kernel has long made extensive use of red-black trees (rbtrees), which are a form of binary tree; this data structure offers fast lookups and the ability to perform insertions and deletions in bounded time. Red-black trees are found in I/O schedulers, graphics drivers, filesystems, the BPF verifier, CPU-scheduler run queues, network protocols, and beyond. One place they have not been found, though, is in programs written to run in the BPF virtual machine. As the complexity of BPF programs grows, though, so does the demand for advanced data structures. The BPF version of the red-black tree, added by Dave Marchevsky, is meant to address this need. 

Within the kernel, data intended to be stored in an rbtree must be stored in a structure that embeds a [`struct rb_node`](<https://elixir.bootlin.com/linux/v6.2/source/include/linux/rbtree_types.h#L5>). The BPF API looks similar in this respect; as described in [the cover letter to the patch set](<https://lwn.net/ml/bpf/20230209174144.3280955-1-davemarchevsky@fb.com/>), the first step is to define a structure to hold the data of interest along with a `bpf_rb_node` structure: 

```
struct node_data {
        	long key;
        	long data;
        	struct bpf_rb_node node;
        };
```

Kernel code will then declare a variable of type `struct rb_root` to hold the root of the tree. The interface for BPF programs looks a bit different; they must declare the root and associate it with the type it contains using a magic macro: 

```
struct bpf_rb_root tree_root __contains(node_data, node);
```

The program must also declare a spinlock and, by storing it into an [array map](<https://docs.kernel.org/bpf/map_array.html>) alongside the `tree_root` variable, associate that lock with the red-black tree. 

Thereafter, there are three functions (as of 6.3) that can be used to work with the rbtree: 

```
void bpf_rbtree_add(struct bpf_rb_root *root, struct bpf_rb_node *node,
        		   	bool (less)(struct bpf_rb_node *a, const struct bpf_rb_node *b));
        struct bpf_rb_node *bpf_rbtree_remove(struct bpf_rb_root *root,
    			struct bpf_rb_node *node);
        struct bpf_rb_node *bpf_rbtree_first(struct bpf_rb_root *root);
```

Adding a node is done with `bpf_rbtree_add()`; the `less()` function is used to compare nodes so that the new node can be properly located within the tree. The first node in a tree can be had with `bpf_rbtree_first()`, and nodes can be removed with `bpf_rbtree_remove()`. It is a fairly bare API for now; more functions can be expected to appear in the future. 

There are a number of interesting aspects to this API, starting with the fact that it exists at all. For a long time, there was only one way to add a complex data structure to BPF: as a [BPF map](<https://www.kernel.org/doc/html/next/bpf/maps.html>). The BPF virtual machine simply did not have the support needed allow programs to directly manipulate data structures of any complexity. That has changed, especially with the addition of [kfuncs](<https://lwn.net/Articles/856005/>) (which allow BPF programs to directly call functions in the kernel), and [better management of pointers](<https://lwn.net/Articles/900749/>). These features, along with the BPF verifier, create an environment that differs considerably from ordinary kernel development. 

For example, as noted above, the program must associate a spinlock with the red-black tree. Once that is done, the verifier will ensure that no access to the tree — or to the data contained within it — happens unless the spinlock is held at the time. Acquiring a spinlock to access a data structure protected by that lock is mandatory in kernel code; severely unpleasant things will happen if the rule isn't followed. But the language itself cannot enforce that rule, so locking bugs are a fairly routine occurrence. As long as the verifier is doing its job properly, similar bugs cannot occur in BPF programs, at least as far as access to an rbtree is concerned. 

The verifier also enforces rules regarding the ownership of pointers. For example, the first step in adding a node to an rbtree will be allocating that node, which is typically done with a call to [`bpf_obj_new()`](<https://git.kernel.org/linus/958cf2e273f0>). On return from that call, the program will own the resulting pointer, and the verifier must be able to convince itself that the program will suitably dispose of that pointer before it exits. One way to do so is to free the memory again, naturally, but that is not hugely interesting. Another is to use `bpf_rbtree_add()` to add the new node to an rbtree, which will then take responsibility for it. If the program removes a node from the tree, it must, once again, take responsibility for disposing of it or the verifier will not let it run. 

In other words, the verifier is implementing a sort of ownership model to ensure that memory leaks do not happen. It can also ensure that, once an object has been freed, the BPF program will no longer attempt to access it. In the case of rbtrees, implementing that check took some work, since accessing the tree (with `bpf_rbtree_first()`, for example) can create "non-owning" references that must all be invalidated when a node is freed. In other words, the verifier is preventing the creation of dangling pointers via aliases to a freed data structure. 

The end result of all this checking is a programming environment that seems a bit more Rust-like and less C-like; there are whole classes of bugs that are eliminated before a program is allowed to run. Making the verifier happy can be a notoriously difficult task for some programming patterns, but the end result should be a higher level of assurance that the program will not damage the kernel. 

Meanwhile, as of this writing, there are no users of the new data structure in the kernel. As a general rule, kernel developers will resist the addition of an API without associated users, but BPF is different; the whole point of a BPF API is to be available to programs that are not part of the kernel tree. There is one known user out of the kernel tree in the form of the [extensible scheduler class](<https://lwn.net/Articles/922405/>); it would not be surprising to see others turn up as well. 

As BPF maintainer Alexei Starovoitov [said](<https://lwn.net/Articles/909095/>) at a conference in 2022, the intent behind the BPF work is to create a safer version of the C language for kernel programming. This work is being done outside of the normal limelight that follows programming-language development and, like much of the kernel, is not following any sort of long-term roadmap. If this effort succeeds, kernel programming in the coming years may be quite different from how it has been until now. The addition of the red-black tree for BPF programs, in other words, is a piece of an interesting and novel forest.
