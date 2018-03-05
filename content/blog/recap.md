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
formats sysfs style documentation. Link to the project -
[abi2doc](https://github.com/aishpant/attribute-documentation).

From a [rough
estimate](https://github.com/aishpant/documentation-scripts/blob/master/result/output.csv)[^1],
there are around 2000 attributes that are undocumented in the kernel. Using
`abi2doc`, I have added documentation for **312** sysfs attributes. A list of my
documentation patches is available
[here](https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/log/?qt=author&q=Aishwarya+Pant).

Plotted[^2] below is a graph of the number of documented attributes in all the stable kernel
versions. The next kernel release 4.16 should have `2867` or more documented attributes compared to
`2442` from Linux 4.15.

<div>
<img src="/images/sysfs_line_plot.jpeg" alt="plot of number of documented attributes per kernel release" style="max-width: 100%;width: 600px;"  width="600" />
</div>

Yep, the steep line at the top are some of my patches!

As I mentioned in a [previous
post](https://aishpant.github.io/blog/attribute-documentation/), the ABI
documentation format looks like the following:  

What: (the full sysfs path of the attribute)  
Date: (date of creation)  
KernelVersion: (kernel version it first showed up in)  
Contact: (primary contact)  
Description: (long description on usage)

`abi2doc` can fill in the `'Date'` and the `'KernelVersion'` fields with high
accuracy. The `'Contact'` details are prompted once per script run, and the
others `'What'` and `'Description'` are prompted on every attribute.

Descriptions are collected from various sources:

* From the commit message that introduced the attribute.

* From the comments around the attribute declaring macro and the attribute
  show/store functions.

* From the structure fields that map to the attribute.

	For eg. consider the attribute declaring macro `PORT_RO(dest_id)` and
	its show function `port_destid_show(...)`.

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

	The attribute show functions typically contain a conversion to a driver
	private struct and then one or many fields from it are put in the
	buffer.

	In the example above, the driver private structure is of type
	`rio_mport` and the attribute `port_id` maps to the field
	`host_deviceid` in the structure.

	```c
  struct rio_mport {
	...
	int host_deviceid;      /* Host device ID */
	struct rio_ops *ops;    /* low-level architecture-dependent routines */
	unsigned char id;       /* port ID, unique among all ports */
	...
	};
	```
	  There's a comment against `host_deviceid` here and this can be
	  extracted using coccinelle.

I was hoping to build something that would fill in all parts of the
documentation i.e. the full attribute path and a concise description. But,
automatic generation of documentation is kind of a fantastical idea.

##### Usage

Prerequisites:

- Coccinelle - [install instructions](http://coccinelle.lip6.fr/download.php)
- Python 3
- Linux Kernel source code

abi2doc is available on [PYPI](https://pypi.python.org/pypi/abi2doc). Install with `pip`:

```bash
pip install abi2doc
```

The library is currently tested against Python versions `3.4+` and intended to
run inside the kernel source.

```bash
usage: abi2doc [-h] -f SOURCE_FILE -o OUTPUT_FILE

Helper for documenting Linux Kernel sysfs attributes

required arguments:
  -f SOURCE_FILE  linux source file to document
  -o OUTPUT_FILE  location of the generated sysfs ABI documentation

optional arguments:
  -h, --help      show this help message and exit
```

Example usage - if I want to document sysfs interface of the lp855x series
backlight driver, I will run the following command:

```bash
abi2doc -f drivers/video/backlight/lp855x_bl.c -o sysfs_doc.txt
```

The script will fill in the `'Date'` and the `'KernelVersion'` fields for found
attributes. The `'Contact'` details are  prompted once per script run, and the
others `'What'` and `'Description'` are prompted on every attribute. The entered
description will be followed by hints, as shown in the generated documentation
below.

```
What:       /sys/class/backlight/<backlight>/bled_mode
Date:       Oct, 2012
KernelVersion:  3.7
Contact:    dri-devel@lists.freedesktop.org
Description:
        (WO) Write to the backlight mapping mode. The backlight current
        can be mapped for either exponential (value "0") or linear
        mapping modes (default).
        --------------------------------
        %%%%% Hints below %%%%%
        bled_mode DEVICE_ATTR drivers/video/backlight/lm3639_bl.c 220
        --------------------------------
        %%%%% store fn comments %%%%%
        /* backlight mapping mode */
        --------------------------------
        %%%%% commit message %%%%%
        commit 0f59858d511960caefb42c4535dc73c2c5f3136c
        Author: G.Shark Jeong <gshark.jeong@gmail.com>
        Date:   Thu Oct 4 17:12:55 2012 -0700

            backlight: add new lm3639 backlight driver

            This driver is a general version for LM3639 backlgiht + flash driver chip
            of TI.

            LM3639:
            The LM3639 is a single chip LCD Display Backlight driver + white LED
            Camera driver.  Programming is done over an I2C compatible interface.
            www.ti.com

            [akpm@linux-foundation.org: code layout tweaks]
            Signed-off-by: G.Shark Jeong <gshark.jeong@gmail.com>
            Cc: Richard Purdie <rpurdie@rpsys.net>
            Cc: Daniel Jeong <daniel.jeong@ti.com>
            Cc: Randy Dunlap <rdunlap@xenotime.net>
            Signed-off-by: Andrew Morton <akpm@linux-foundation.org>
            Signed-off-by: Linus Torvalds <torvalds@linux-foundation.org>
```

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
engineer roles and I can work remotely. It would be quite amazing if I get to
work on open source. Fun fact - I have not been through a job interview in 3
years, so it looks like I am going to have some fun :-)

[^1]: The link contains an analysis of undocumented attributes only in the `driver` source files. Possibly outdated.
[^2]: For anyone who is interested, [this](https://github.com/aishpant/documentation-scripts/blob/master/stats.sh) is the script I used to calculate the number of documented attributes and plotted the graph using [Plotly](https://plot.ly/).
