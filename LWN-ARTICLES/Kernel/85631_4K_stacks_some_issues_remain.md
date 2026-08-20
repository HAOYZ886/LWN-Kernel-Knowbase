---
title: "4K stacks: some issues remain"
url: https://lwn.net/Articles/85631/
date: "May 18, 2004"
category: Kernel stack
author: 
---

[Last week's Kernel Page](<https://lwn.net/Articles/83794/>) talked about the push toward 4K stacks on the i386 architecture. While most of the problems with the smaller stack size have been worked out, a few remain. Witness, for example, [this problem report](<https://lwn.net/Articles/85632/>); it would appear that the 2.6.6 Radeon framebuffer driver is overflowing the 4K stack. 

The problem was quickly [narrowed down](<https://lwn.net/Articles/85633/>) to a couple of new fields added to the `radeon_regs` structure: 

```
struct radeon_regs {
            ....
            u32             palette[256];
            u32             palette2[256];
    };
```

If one of these structures is placed on the kernel stack (as happens in the radeonfb driver), those two arrays, by themselves, take half of the available space. If that weren't sufficiently annoying, there is the little fact that those arrays are part of an ongoing development and are not actually used for anything in 2.6.6. 

Fixing this particular problem is relatively easy, but this episode has reawakened interest in finding large stack users automatically. One never knows when a developer will expand a data structure without realizing that it is used on the stack in some other place; rather than letting users find this sort of mistake the hard way, it would be better to look for them explicitly earlier in the development process. To that end, several scripts have been posted which seek out large stack users in a compiled Linux kernel. A quick look at these scripts makes it clear that kernel code is, by no means, the scariest code out there: 

```
objdump --disassemble "$@" | \
    sed -ne '/>:/{s/[<>:]*//g; h; }
     /subl\?.*\$0x[^,][^,][^,].*,%esp/{
     s/.*\$0x\([^,]*\).*/\1/; /^[89a-f].......$/d; G; s/\(.*\)\n.* \(.*\)/\1 \2/; p; };
     /subl\?.*%.*,%esp/{ G; s/\(.*\)\n\(.*\)/Dynamic \2 \1/; p; }; ' | \
     sort | \
    perl -e 'while (<>) { if (/^([0-9a-f]+)(.*)/) { $decn = hex("0x" . $1);\
         if ($decn > 400) { print "$decn $2\n";} } }'
```

(from [a script](<https://lwn.net/Articles/85634/>) by Keith Owens and Arjan van de Ven). Several variants have been posted, most of which are trying to support multiple architectures. None yet have solved the full problem, however: finding full call chains whose cumulative stack usage exceeds the space available. With or without that feature, some sort of stack usage checker is likely to be merged into the kernel build system before too long. That should help the developers to trap the most obvious problems before they find their way into a released kernel.
