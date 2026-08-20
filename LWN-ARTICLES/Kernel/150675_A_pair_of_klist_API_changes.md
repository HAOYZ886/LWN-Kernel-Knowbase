---
title: A pair of klist API changes
url: https://lwn.net/Articles/150675/
date: "September 7, 2005"
category: klist
author: 
---

The klist type implements a linked list with built-in locking; it was described here [last March](<http://lwn.net/Articles/128644/>). The 2.6.14 kernel will contain a couple of API changes affecting klists. 

The first is a simple change for a couple of klist functions, which now have the following prototypes: 

```
void klist_add_head(struct klist_node *node, struct klist *list);
        void klist_add_tail(struct klist_node *node, struct klist *list);
```

The change is that the order of the two parameters has been switched. This change makes the klist functions use the same ordering as the older `list_head` functions, hopefully leading to a lower level of programmer confusion. 

The more complicated change has to do with reference counting. The klist list iteration functions can hold references to objects on the list, but the higher level code (which actually creates the objects) does not know about those references. Somehow, the klist code must be able to tell the next layer up about references it holds during list iteration. To that end, `klist_init()` has picked up a couple of new parameters: 

```
void klist_init(struct klist *list, void (*get)(struct klist_node *node),
    		    void (*put)(struct klist_node *node));
```

The `get()` and `put()` functions are a bit of glue code which allows the klist code to take and release references. All code using klists must now provide these functions at initialization time.
