---
layout: post
title: Notes on "The Linux Device Model"
date: 2021-02-23 19:00:00 +0900
---

As part of [The Eudyptula Challenge][eudyptula], Little tasked me to read Chapter 14 of
[Linux Device Drivers][ldd3], titled *The Linux Device Model*. Since the introduced topics are
pretty advanced, I took some notes along the way. These skip many details, so if you find the topic
interesting, please go read the whole chapter. Note that this book is a bit out of date since it
was written for kernel version `2.6.10`.

The Linux device model can be roughly described as a data structure used to represent the hierarchy
and relationships between buses, drivers, and devices in a Linux system. This model is exposed and
can be interacted with, using the *sysfs* pseudo filesystem, which is usually mounted at `/sys`.

### 🧱 Kobjects
Kobjects are the basic building blocks of the Linux device model. They can be used for:
- reference counting (embedded inside other data structures)
- *sysfs* representation
- device model representation
- hotplug event handing

We won't go into detail about reference counting, but kobject's reference counter can be modified
using the `kobject_get()` and `kobject_put()` functions. Once the counter reaches `0`, the kobject
is freed.

All kobjects, if exposed, show up in `/sys` as directories. Inside these directories reside the
kobject's attributes as files. Each kobject is assigned a *ktype*, which defines its default
attributes, along with `show()` and `store()` functions (through `sysfs_ops`). These verify and
implement the read/write operations on the attributes (files). If you wish to modify non-default
attributes to a kobject, you can use the `sysfs_{create,remove}_file()` functions.

#### Ksets
While relationships between kobjects can be defined using the `parent` field, they are usually
grouped into *ksets* (using a linked list). These act as a container for other kobjects and ksets,
and always appear under `/sys`, through an embedded kobject. If a kobject is part of a kset, it uses
the parent kset's ktype. Kset membership can be modified using the `kobject_{add,del}()` functions.

Kobjects without a parent will appear in the *sysfs* root, but this is rarely desired. On the
other hand, ksets must belong to a *subsystem*. A subsystem is a representation of a part of the
device model (for example the device subsystem under `/sys/devices` or the PCI subsystem under
`/sys/bus/pci`). The `subsystem` data structure contains a single kset and a semaphore to control
concurrent access during kset traversal.

While the *sysfs* filesystem has a tree structure, additional relationships between kobjects are
represented using symbolic links. For example, the device a driver is attached to is simply a
symbolic link to `/sys/devices/<device>`.

Each time a kobject is created or removed, a hotplug event is triggered, which allows the userspace
to react appropriately and load the necessary drivers. This used to be handled by the
`/sbin/hotplug` script. The events bubble up the device model, and each layer can add the
corresponding environment variables or suppress the event if necessary.

Here's an example of a simple *sysfs* hierarchy:
```bash
$ tree /sys
/sys
├── kobject
├── kset
│   ├── kobject1
│   │   ├── attribute1
│   │   └── attribute2
│   ├── kobject2 -> ../subsystem/kobject
│   └── kobject3
└── subsystem
    └── kobject
```

### 🏠 Devices and drivers
Using kobjects as a base, higher-level data structures are fairly similar. They each use an embedded
kobject and represent a small part of the Linux device model. Whenever your functions are passed a
lower-level object, which is embedded in a custom data structure, you can use the `container_of()`
macro to get the containing data structure defined in your kernel module.

Similar to kobjects, these data structures all have their own functions for initialization and
attribute handling. For ease of development, convenience macros like `DEVICE_ATTR()` are defined.

#### Buses
Buses are used as a channel between the CPU and multiple devices. While not necessarily representing
a physical bus, all devices must be connected via a bus. Virtual buses are called "platform" buses.
Note that each bus is its own subsystem and is located under `/sys/bus`. Each bus contains a
`devices` and a `drivers` kset. As mentioned before, devices are symbolic links to `/sys/devices`.

Whenever a new device or a driver for a specific bus is added, the `match()` function is called,
which tries to assign the device with a driver. After the function finds a driver, the kernel calls
its `probe()` method, which performs additional checks and initialized the device. If `probe()`
returns an error, `match()` continues with other drivers.

#### Devices
Every device in a Linux system is represented by a `device` structure. It contains all the fields
needed to identify a device and its location within the device tree, while most other fields are
initialized by the specified subsystem, which assigns the bus and a driver. These subsystems
usually wrap the `device` structure to track additional information. As mentioned, all devices are
located in `/sys/devices`.

Classes are an even higher-level abstraction over devices, based on what a device does (for example
storage or input devices). They have pretty a complex interface, so we won't go into detail.
Classes are usually located in `/sys/class`, which mostly contain symbolic links to devices. Class
membership is handled by higher-level subsystem code.

#### Drivers
Drivers, usually located under `/sys/bus/<name>/drivers`, implement the logic to interact with a
device. Once it's registered with a subsystem, it can be matched with devices connected to a system.
The structure of a `driver` object is similar to `device`, but it also contains various methods to
initialize and shutdown devices. One of these methods is `probe()`, mentioned above.

This concludes my notes about the various data structures in the Linux device model. For an example
of how this all works in practice in the PCI subsystem, please check the *Putting It All Together*
section in the book. The [explanation][chapter14] there is far better than what I could summarize here.

### 📚 References
- [Linux Device Drivers, Third Edition][ldd3]
- [Everything you never wanted to know about kobjects, ksets, and ktypes][kobject-docs]
- [A fresh look at the kernel's device model][fresh-look]

[eudyptula]: http://eudyptula-challenge.org
[ldd3]: https://lwn.net/Kernel/LDD3/
[chapter14]: https://static.lwn.net/images/pdf/LDD3/ch14.pdf
[kobject-docs]: https://www.kernel.org/doc/html/latest/core-api/kobject.html
[fresh-look]: https://lwn.net/Articles/645810/
