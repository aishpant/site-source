---
title: "attribute documentation : what, why, how?"
date: 2018-01-03T15:01:59+05:30
tags: ["outreachy", "documentation", "linux", "sysfs"]
slug: "attribute-documentation"
---

Before I talk about the title of the post, I will take a diversion to try to
explain what kernel hackers mean when they say they [do not break
userspace](https://lkml.org/lkml/2012/12/23/75).

#### What is an API and an ABI?

API: Application Programming Interface  
ABI: Application Binary Interface

The two terms are closely related and can be a source of confusion. Both APIs
and ABIs specify how two components interact with each other. The boundary for
API interaction is source code and for ABI it is binary executables.

An API might be something you are more familiar with. For eg. the public methods
and data structures of a Java class.

An ABI is hardware and platform dependent. This is the definition from [The Linux
Programming Interface](http://man7.org/tlpi/): Among other things, an ABI
specifies which registers and stack locations are used to exchange this
information, and what meaning is attached to the exchanged value. Once compiled
for a particular ABI, a binary executable should be able to run on any system
presenting the same ABI.

The one above is a more general definition. The **Linux ABI** defines how
userspace communicates with the kernel - `syscalls`, pseudo file systems (`procfs`,
`debugfs`, `configfs`, `sysfs`), ioctls, etc. When the kernel changes its ABI (a new
release or version update), all userspace applications and third-party modules
that interact with it need to be recompiled. No patch is allowed to
intentionally break the userspace. An unstable change could be addition of a new
argument to a syscall or changing the behaviour of an existing argument.
Unfortunately any such break is usually detected after a release (when userspace
applications are already using it).

What can cause undefined behaviour? Incomplete testing and incomplete
specification. A documented behaviour can help users other than developers to
test new interfaces. And this is why **both testing and documentation efforts are
essential for kernel development**.

Since breaking userspace is not acceptable, backward compatibilty is guaranteed
for 2 years for interfaces documented in [`Documentation/ABI/stable`](https://www.kernel.org/doc/Documentation/ABI/README).

## sysfs file system

> [A web woven by a spider on drugs](https://lwn.net/Articles/31185/)

Now that we all agree that documentation is important, we come to the in-memory
virtual filesystem `sysfs`. It is automatically mounted at `/sys`.

```bash
# ls -l /sys

total 0
drwxr-xr-x   2 root root 0 Dec 26 21:36 block
drwxr-xr-x  39 root root 0 Dec 25 19:42 bus
drwxr-xr-x  57 root root 0 Dec 25 19:42 class
drwxr-xr-x   4 root root 0 Dec 25 19:42 dev
drwxr-xr-x  21 root root 0 Dec 25 19:42 devices
drwxr-xr-x   5 root root 0 Dec 25 19:42 firmware
drwxr-xr-x   7 root root 0 Dec 25 19:42 fs
drwxr-xr-x   2 root root 0 Dec 25 19:42 hypervisor
drwxr-xr-x  11 root root 0 Dec 25 19:42 kernel
drwxr-xr-x 200 root root 0 Dec 25 19:42 module
drwxr-xr-x   2 root root 0 Dec 25 19:42 power
```

The directories under `sysfs` export information about the internal kernel data
structures and layout. It provides a hierarchial view of the device structure
under `/sys/devices`. Each directory in `sysfs` is a [kobject](https://www.kernel.org/doc/Documentation/kobject.txt) (kernel object) and
the files in a directory are the attributes of a kobject.

What are kobjects? You actually don't need to work with or know internals of
kobjects, or ksets or kobj\_type. These are fundamental objects (like base
classes of Object Oriented Programming) emebedded in other high level objects,
that have a name and provide reference counting at the least. kobjects are
embedded in devices, drivers, subsystems, filesystems. They are everywhere!

The **goal of sysfs** is [to do one thing and do it
well](https://en.wikipedia.org/wiki/Unix_philosophy).

* `sysfs` attributes should export one value per file.
* Values should be text-based and map to simple C types. It was created
  to avoid the highly structured or highly messy representation of `procfs`.

Unlike `syscalls` which are generally stable, forming the bulk of the userspace
ABI and well-documented through the [linux manpages
project](https://www.kernel.org/doc/man-pages/), it is possible for `sysfs`
interfaces to change in an incompatible way between release versions.  There are
around 332 `syscalls`[^1] whereas I can see about 30,000 `sysfs` files[^2] on my
laptop. **This is massive**, having only been introduced in kernel release
version 2.5 with the unification of the device model.

In summary:

* The `sysfs` reflects the kernel's internal device model.
* It lives in memory.
* What you see is not real files! They don't exist on your disk.
* Subsystems like class, bus, block etc are just grouping of devices by
  connection to bus type, functionality, device type etc.
* Redundancy is avoided by using symlinks.
* This ABI is huge! Which also makes it difficult to maintain :-(

```bash
# ls -l /sys/class/hwmon

total 0
total 0
lrwxrwxrwx 1 root root 0 Dec  21 17:02 hwmon0 -> ../../devices/virtual/hwmon/hwmon0
lrwxrwxrwx 1 root root 0 Dec  21 17:02 hwmon1 -> ../../devices/virtual/hwmon/hwmon1
lrwxrwxrwx 1 root root 0 Dec  21 17:02 hwmon2 -> ../../devices/platform/coretemp.0/hwmon/hwmon2
```
Example of symlinking. Hardware monitoring devices are grouped by functionality under
/sys/class/hwmon and symlinked to entries under /sys/devices.

#### What is my project about?

There are many configurable parameters in the sysfs ABI, the attributes or
virtual files can be read or written to from userspace. The documentation for
them resides in Documentation/ABI. But many of them are not represented. From a
preliminary study[^3], I found that ~2000 attributes are documented and ~1000
attributes are missing.

The ABI documentation format looks like the following:

_What_:		(the full sysfs path of the attribute)  
_Date_:		(date of creation)  
_KernelVersion_:	(kernel version it first showed up in)  
_Contact_:	(primary contact)  
_Description_:	(long description on usage)  

The goal is to use scripting tools like
[coccinelle](http://coccinelle.lip6.fr/:Coccinelle) to collect information about
various parts that are needed to fill this.

How can this be filled in?

1. The Date, KernelVersion can be extracted from the commit that introduced it.
   Date is the author commit date and KernelVersion is the first release version
   tag that contains the commit.
2. The Contact person may perhaps be the module author but could also be a
   subsystem specific mailing list (mm, rtc, usb).
3. The full attribute path and the description are the hard parts to fill.  The
   Description can be in a commit message or documented along with the driver
   (but not in the ABI). Some hints can be in the code surrounding the attribute
   declaration. Or in the description of the field that it might map to in a
   structure.
4. The official path for attributes, the field What, varies in every subsystem.
   Some prefer a class path or a bus path or a devices one. There are only a few
   ways to create a class device attribute (by declaring a struct class or using
   class\_create()). Again, coccinelle should be helpful here.

Want to know more?

1. [Kernel Recipes 2016 - Linux Device Model - Greg KH](https://www.youtube.com/watch?v=AdPxeGHIZ74)
2. [How to create a sysfs file correctly (2013)](http://kroah.com/log/blog/2013/06/26/how-to-create-a-sysfs-file-correctly/)
3. This is a bit of ancient history: [The Linux Device Model - Chapter 14 Linux Device Drivers (2005)](https://static.lwn.net/images/pdf/LDD3/ch14.pdf)

[^1]: Look at [`arch/x86/entry/syscalls/syscall_64.tbl`](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl) in the source code.
[^2]: I don't know if this deduplicates symlinks: `find /sys -type f | wc -l`
[^3]: I used a cocci script to identify all the standard attribute declarers (`DEVICE_ATTR, DEVICE_ATTR_{RO/RW/WO}`) using the already documented attributes. Then, I used another script to identify all declared attributes using the previously found macros and the position where the attributes are present.
