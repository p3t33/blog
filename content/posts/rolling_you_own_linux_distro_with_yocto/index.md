+++
title = 'Rolling you own Linux distro with yocto'
date = 2024-10-27T11:00:32+03:00
draft = false
tags = ["Linux", "yocto", "bitbake", "poky"]
+++

![](1.png)

This post is the first in a series that will cover embedded development from the perspective of
OS development, starting with the need of such solution, the moving parts, the basic configuration that
can be used on a raspberry pi and anything else that might be relevant.

When it comes to starting an embedded project, one that uses more then a simple micro controller, a choice
how much control over its software components will need to be made, meaning control over the
OS(Linux in the context of this post) and the added logic/responsibility during development.
Such control will come at the expense of development velocity(mostly at the very start of it),
with the control requirements from a hubby project/POC are different then the once from production
project. The former is one of a kind you can tinker with until it works and is thrown into the bowels
of your basement server rack while the latter needs to be easily deployed, scaled, developed, patched and
maintained.

## Picking Off the shelf image
A good place to start is with off the shelf development board such as the Raspberry pi, that has
myriad of choices when it comes to premade images to flash. Each such image is
a Linux distribution with a vendor or community backing it. And at first glance, this looks like the right
call. It is a very fast way to start rolling with almost zero learning curve. This path comes with very
no control over what constitutes as "image", as it is decided for you by the image vendor.

But what this lack of control means and how it impacts development?
- Complete dependency on the OS vendor(and the policy imposed by him), with no control over what
  goes into the image. Or any control over the end of life of the firmware.
- As an expansion on previews point, the image most likely targeting general use, won't be
  cross-development-friendly (compile on the device only which is slow), and not kernel patched
  for real-time performance but will have bunch of tolls you don"t need which will take CPU and storage
  resources.
- Limited selection of popular boards, which might not suite the project needs when it comes to
  coast or hardware.
- Firmware flashing becomes a two step process the project software needs to be installed in a
  separate step once the vendor firmware is flashed and running.
- Sharing firmware parts(OS and application) between devices from the same family with slightly different
  boards becomes impossible as "one size" doesn't fit all.

Because of those Limitations the off the shelf approach can take us only so far. For devices that go
into production full control over the components of the firmware is required. So The next logical
step is to build a custom firmware where you fully control the OS and what goes into it and where
everything including your logic is installed in a single firmware image in a single step.


## Building from scratch
So the reason that premade images are so popular is because creating your own custom Linux image is
not the same as building a single small project from the source this is a herculean task that a simple Cmake
just can't handle(and I implore you to try do the [linux from scratch project](https://www.linuxfromscratch.org/) to
get a taste of what it takes to build your Linux image from scratch). In broad strokes when building your own firmware image you will need to download
the source for Linux kernel and bootloader, download the source for a bunch of libraries and applications.
Patch, configure and compile everything. put all the output files into a staging area that looks like a
Linux root file system. And finally, convert everything to fit into partitions your board will be able to
boot. And then you are still left with configuring all of this for multiple devices which may use
different architecture and drivers. So a build system for creating custom embedded Linux distributions
needs to be defined.

## Defining a build system for custom images
- Provide a mechanism for building images based on configurations (infrastructure as a code), so definition of
images will be highly modular, with software encapsulated by functionality(hardware/BSP, user applications...) in such a
way that each module can be removed and added as a piece of LEGO.
- Reproducible, so the build process is fully transparent to the user.
- Allow for the execution of tasks(get source code for the library, compiling user application, apply patches...) in parallel while
arranging tasks in such a way that task will be order into a queue based on dependencies.
- Provide license management and stop the build if an incorrect license is used.
- It is not an IDE, it is not intended for iterative development(too slow) only to create images to be flashed to a device once all the
software wrinkles are ironed out.
- For development, a cross-compile toolchain is provided as part of the SDK.
- It is not a distribution tool, it is intended for creating Linux images.

## yocto, and why to use it?
yocto is the build system that meets the above definitions. And although yocto, Poky, and
OpenEmbedded are terms that sometimes used interchangeably it is worth clarify the differences
before we go farther.

OpenEmbedded project set out to define a build system for GNU/Linux based custom images. By using a set of
binary packages, which could then be combined in various ways to create a target system It did this by
creating recipes for each piece of software(bootloader, kernel...) and using bitBake as its task scheduler.

It is highly flexible, and by supplying the metadata(recipes) an entire image to your own specification
can be created. The project's main goal is to provide a comprehensive set of metadata for a wide variety
of architectures and not to be a reference distribution designed to be the foundation for others.

The OpenEmbedded project predates the yocto projects and in a sense can be considered as “upstream” for
yocto. A fork of OpenEmbedded was created with the goal to have a more conservative choice of packages
and to create releases that were stable over a period of time. The fork was named Poky. Later Poky was
transferred to the Linux Foundation and yocto Project was born. Yocto to Poky is what Canonical to Ubuntu.

Poky has a dual definition

It is a build system for custom images(by combining metadata + bitbake) and in this sense, it
isn't a single "GNU/Linux distribution" but a tool to create user-defined  images.
It is a subset of OpenEmbedded, it uses its tools, and uses some of the software components in the
form of OpenEmbedded-Core as its basis. It creates from source Bootloader, Kernel, rootfs, other
software-defined by the user, and creates firmware images. Poky is platform-independent and performs
cross-compiling.

Although the Yocto project states "It isn't an embedded Linux distribution - it creates a costume one
for you". Poky is in fact also a reference distribution. The Poky distribution is made using components
from OpenEmbedded, demo BSPs(BeagleBone Black, generic x86-64...), helper scripts to easily set up a build
environment, QEMU emulator to test the image, and the bitbake task scheduler. This makes poky a
ready-to-cook subset of OpenEmbedded (OE) that helps users to understand the build system and to
create their own image possibly based on Poky.


Pros:
- Widely supported by semiconductor vendors with an active developer community.
- It is organized into separate layers(for software encapsulates )which can move at
  their own pace with separate release schedules. Which makes it highly customizable and expandable.
- By providing SDK it creates separations between kernel/user space development.
- Firmware images defined by the project are architecture agnostic.
- Optimized for performance by utilizing smart dependency management, the parallelism of task
  execution, and the use of cache.

Cons:
- Overkill for one time hubby projects.
- Steep learning curve
- Unfamiliar environment to non-embedded developers
- Resource-intensive in the form of long initial build times and use of large disk space.


