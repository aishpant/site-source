---
title: "outreachy : closing notes"
date: 2018-02-26T22:15:34+05:30
tags: ["outreachy", "documentation", "linux", "sysfs"]
slug: "outreachy-recap"
---

This is the last week of my three-month internship with
[Outreachy](http://outreachy.org/).

##### Did I accomplish what I had set out to do?

In some ways, yes. I have written a helper tool that tries to generate and
formats sysfs style documentation. Link to the project - [attribute
documentation](https://github.com/aishpant/attribute-documentation).

As I mentioned in a [previous
post](https://aishpant.github.io/blog/attribute-documentation/), the ABI
documentation format looks like the following:  

What: (the full sysfs path of the attribute)  
Date: (date of creation)  
KernelVersion: (kernel version it first showed up in)  
Contact: (primary contact)  
Description: (long description on usage)

The scripts under this project can fill in the 'Date' and the 'KernelVersion'
fields with high accuracy. The 'Contact' details is prompted for once, and the
others 'What' and 'Description' are prompted for every attribute.

Descriptions are collected from various sources:

* From the commit message that introduced the attribute.

* If suppose documentation is present somewhere, plain old grep for the
  attribute name in the Documentation folder.

* Comments around the attribute declaring macro and the attribute show/store
  functions.

* From the structure fields that map to the attribute.

  For example - the attribute show functions usually look like this:
```c
        static ssize_t
        port_destid_show(struct device *dev, struct device_attribute *attr,
                         char *buf)
        {
                struct rio_mport *mport = to_rio_mport(dev);

                if (mport)
                        return sprintf(buf, "0x%04x\n", mport->host_deviceid);
                else
                        return -ENODEV;
        }
```
  There is a conversion to a driver private struct and then one or many fields from
  it are put in the buffer.

  In the example above, it's a struct of type `rio_mport`.
```c
  struct rio_mport {
        ...
        int host_deviceid;      /* Host device ID */
        struct rio_ops *ops;    /* low-level architecture-dependent routines */
        unsigned char id;       /* port ID, unique among all ports */
        ...
        };
```
  There's a comment against `host_deviceid` here and this can be extracted using
  coccinelle.

Using these scripts, I have added documentation for **312** sysfs attributes. I
hope to add some more in the coming week. A list of my documentation patches is
available
[here](https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/log/?qt=author&q=Aishwarya+Pant).

<div>
<img src="/images/sysfs_line_plot.jpeg" alt="plot of number of documented attributes per kernel release" style="max-width: 100%;width: 600px;"  width="600" />
</div>

Yep, the steep line at the top are some of my patches! These should be a part of
the 4.16 kernel release.

I was hoping to build something that would fill in all parts of the
documentation. But, automatic generation of documentation is kind of a
fantastical idea.

##### Usage

Prerequisites:

- Coccinelle - [install instructions](http://coccinelle.lip6.fr/download.php)
- Python 3
- Linux Kernel source code

First, you need to clone the GitHub
[project](https://github.com/aishpant/attribute-documentation). The `make doc`
command needs two options to run - path to the kernel source file or directory
which needs documentation, and the path to the kernel source code base.

```bash
$ make doc FILE=$(source file or directory) KERNEL_PATH=$(path to kernel source)
$ make clean # clean temporary & generated files
```

Example usage - if I want to document sysfs interface of the lp855x series
backlight driver, I will run the following command:

```bash
$ make doc FILE=~/projects/linux/drivers/video/backlight/lp855x_bl.c KERNEL_PATH=~/projects/linux
```

While `make doc` runs, it will create a `description.txt` file containing
possible descriptions for the attributes. You will need to refer to it while
filling in the documentation. The formatted documentation is appended to a file
named `sysfs_doc`.

##### What did I learn?

**Documentation within the kernel is kind of a mess.** You are expected to read
[Linux Device Drivers book](lwn.net/Kernel/LDD3/) to even begin to understand
the kernel. Sadly, this book is outdated since it was written in 2005 and there
seems to be [no
plan](https://www.reddit.com/r/linux/comments/61q6y8/what_happened_to_linux_device_drivers_4th_edition/dfgkzqz/)
for a fourth edition. Most in-kernel documentation is quite sparse, sometimes
outdated, assumes some prior knowledge and expects readers to go through
specifications or datasheets. This is a **huge** barrier for newbies but it is
also true that maintainers really appreciate documentation patches.

**It made me think about how to write and communicate effectively.** Often while
writing, I would ask myself these questions - Is this documentation easy to
follow? Am I using unexplained abbreviations? Is this information complete? Am I
skipping describing something because it seems 'obvious'?

**Overview of kernel subsystems**. I learned how real time clocks worked, about
cpuidle (power management for idle states), the loop block driver, the backlight
drivers, and the network communications standard used in high-performance
computing (infiniband). The kernel is **huge** and there are lot of exciting
things to learn.

**Working remotely is hard**. It took me a long time to get used to it. My
mentor and I had weekly meetings where we decided on the tasks for the week. I
may or may not have always finished them on time! There are many benefits of
working from home but it's odd not being around people all day.

**I am the maintainer of an open source project now!** Sort of. There is a lot
more to be done in [attribute
documentation](https://github.com/aishpant/attribute-documentation). You can
help with kernel documentation - from a rough estimate, there are around 2000
attributes that are undocumented. Or you can help me improve the python scripts.
This project has a README and there are some newbie friendly tasks too.

**I am a much more confident engineer and a better programmer** because of my
Outreachy experience. I don't worry too much if I encounter something new
anymore. I realise that programming languages, technical concepts can be
mastered over time and willingness to learn is a good skill to have.

I worked on improving documentation for one of the most important software
projects and got paid while doing it! I am grateful to the Outreachy organisers
for making this happen, and my mentor Julia Lawall for guiding me over the last
few months.

##### What's next?

I am looking a job! I am interested in production / systems / infrastructure
engineer roles. It would be quite amazing if I get to work on open source. Fun
fact - I have not been through a job interview in 3 years, so it looks like I am
going to have some fun :-)
