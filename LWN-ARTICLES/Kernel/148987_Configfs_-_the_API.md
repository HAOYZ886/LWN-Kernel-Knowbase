---
title: "Configfs - the API"
url: https://lwn.net/Articles/148987/
date: "August 24, 2005"
category: Configfs
author: 
---

The [configfs introduction](<https://lwn.net/Articles/148973/>) described how this filesystem looks from user space. Anybody wanting to use configfs within a kernel subsystem will also be interested in the kernel-side interface. The configfs API will be somewhat familiar to developers who have worked with kobjects and sysfs; there are some differences, however. What follows is a blindingly fast overview of the configfs API; hold on tight. 

Configfs implements a set of object types used to put together a configuration hierarchy: 

  * A **config item** (`struct config_item`) is the internal representation of an object to be configured. It corresponds to a directory in user space, and behaves somewhat like (the sysfs aspect of) a kobject in kernel space. Each config item has one or more **attributes** , represented in user space as files containing text values. Like kobjects, config items are almost always embedded within other, domain-specific structures. 

  * A **config group** (`struct config_group`) is just a config item which can contain other config items (or groups). 

  * A **config subsystem** (`struct configfs_subsystem`) is a top-level config group. Like the sysfs `subsystem` type, it contains a semaphore used for mutual exclusion within the configfs code. The presence of the semaphore is somewhat interesting; the sysfs equivalent has been recognized for a while as being superfluous, and it will eventually be eliminated. The system being configured will have to perform its own internal locking anyway, so the same lock might as well be used at the configfs level. 

More specifically, anybody wanting to create a configfs hierarchy must set up one or more config items - even if the only item, at the outset, is the `config_subsystem` structure implementing the top-level directory. Creating a config item requires, in turn, that some other structures be set up. The first of these is: 

```
struct configfs_item_operations {
    	void (*release)(struct config_item *item);
    	ssize_t (*show_attribute)(struct config_item *item,
    				  struct configfs_attribute *attr,
    				  char *buffer);
    	ssize_t (*store_attribute)(struct config_item *item,
    				   struct configfs_attribute *attr,
    				   const char *buffer, size_t size);
    	int (*allow_link)(struct config_item *src,
    			  struct config_item *target);
    	int (*drop_link)(struct config_item *src,
    			 struct config_item *target);
        };
```

This structure defines how a specific config item operates. The `release()` method will be called whenever a config item's reference count drops to zero; its job is to perform the necessary cleanup. Attributes are implemented via the `show_attribute()` and `store_attribute()` methods, which work in the obvious manner. The final two methods, if present, control whether the creation of symbolic links between config items is allowed (`allow_link()`) and provide notification when a symbolic link is removed (`drop_link()`). 

The above operations structure should be filled in for a specific type of config item. Then, it is necessary to store a pointer to the structure in a `config_item_type` structure: 

```
struct config_item_type {
    	struct module                    *ct_owner;
    	struct configfs_item_operations  *ct_item_ops;
    	struct configfs_group_operations *ct_group_ops;
    	struct configfs_attribute        **ct_attrs;
        };
```

Here, `ct_owner` is used to manage module reference counts, and `ct_item_ops` is the set of methods seen above. `ct_group_ops` is a separate set of operations for config groups; we'll get to those shortly. The final field, `ct_attrs`, defines the actual attributes which belong to this type of config item; it is an array of pointers to `configfs_attribute` structures: 

```
struct configfs_attribute {
    	char          *ca_name;
    	struct module *ca_owner;
    	mode_t         ca_mode;
        };
```

As with sysfs, the structure representing an attribute contains little information beyond its name and permissions. A single set of functions is used for all attributes belonging to a given item type; they must figure out which attribute is being accessed themselves by looking at the name or by embedding the `configfs_attribute` structure inside another structure. 

An actual config item looks like this: 

```
struct config_item {
    	char                    *ci_name;
    	char                    ci_namebuf[UOBJ_NAME_LEN];
    	struct kref             ci_kref;
    	struct list_head        ci_entry;
    	struct config_item      *ci_parent;
    	struct config_group     *ci_group;
    	struct config_item_type *ci_type;
    	struct dentry           *ci_dentry;
        };
```

Code creating a config item should zero the entire structure, then initialize it with one of: 

```
void config_item_init(struct config_item *item);
        void config_item_init_type_name(struct config_item *item,
    				    const char *name,
    				    struct config_item_type *type);
```

If the name and type are set using the second form, no other initialization is required. The item, once created, will show up in configfs and will contain the attributes defined by its type. 

Config items have a reference count, which is manipulated with the usual sort of functions: 

```
struct config_item *config_item_get(struct config_item *item);
        void config_item_put(struct config_item *item);
```

Items are created within a config group, defined by this structure: 

```
struct config_group {
    	struct config_item	  cg_item;
    	struct list_head	  cg_children;
    	struct configfs_subsystem *cg_subsys;
    	struct config_group	  **default_groups;
        };
```

As noted before, a config group is just a config item which can contain other items (or groups). So it has a `config_item` structure embedded within it. There is also a set of subgroups which will automatically be created whenever a group is created within this group. This list (`default_groups`), along with the list of attributes associated with the config item, define the full contents of the group's directory when it is created. 

Groups are initialized in a manner similar to items: 

```
void config_group_init(struct config_group *group);
        void config_group_init_type_name(struct config_group *group,
    				     const char *name,
    				     struct config_item_type *type);
```

Groups must define a set of group operations (and store a pointer to them in the `config_item_type` structure): 

```
struct configfs_group_operations {
    	struct config_item *(*make_item)(struct config_group *group,
    					 const char *name);
    	struct config_group *(*make_group)(struct config_group *group,
    					   const char *name);
    	int (*commit_item)(struct config_item *item);
    	void (*drop_item)(struct config_group *group,
    			  struct config_item *item);
        };
```

Any particular config group type should only define either `make_item()` or `make_group()`, but not both. If `make_group()` exists, it will be called in response to a request from user space to create a directory; its job is to create a `config_group` structure, initialize it, and return it. In the absence of a `make_group()` method, `make_item()` will be called instead. There is, thus, no way to create a group which allows the dynamic creation of both items and groups within it; that limitation is unlikely to be a problem in most cases. 

The `drop_item()` method will be called when an item (or group) is deleted from the group. The `commit_item()` method is there to support transactional access to group members; that functionality is not implemented in the current configfs patch. 

The top level of the hierarchy is a `configfs_subsystem` structure, which is just a special group: 

```
struct configfs_subsystem {
    	struct config_group	su_group;
    	struct semaphore	su_sem;
        };
```

Code creating a subsystem must first initialize the embedded group in the usual manner, then register the subsystem with: 

```
int configfs_register_subsystem(struct configfs_subsystem *subsys);
```

There is a `configfs_unregister_subsystem()` function as well. 

The above whirlwind tour is, hopefully, enough to give a feel for how to work with configfs. Those wanting more information may wish to consult the extensive [documentation file](<https://lwn.net/Articles/149005/>) and the [example module](<https://lwn.net/Articles/149007/>) distributed with the configfs patch.
