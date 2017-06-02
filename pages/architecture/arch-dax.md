---
title: DAX
sidebar: architecture_sidebar
permalink: arch-dax.html
folder: architecture
---

Clear Containers utilises the Linux kernel DAX (Direct Access filesystem)
feature to efficiently map some host side files into the guest VM space.
In particular, Clear Containers uses the `QEMU` nvdimm feature to provide a
memory mapped virtual device that can be used to DAX map the mini-OS root
filesystem into the guest space.

Mapping files using DAX provides a number of benefits over more traditional VM
file and device mapping mechanisms:

- Mapping as a direct access devices allows the guest to directly access
the memory pages (such as via eXicute In Place (XIP)), bypassing the guest
page cache. This provides both time and space optimisations.
- Mapping as a direct access device inside the VM allows pages from the
host to be demand loaded using page faults, rather than having to make requests
via a virtualised device (causing expensive VM exits/hypercalls), thus providing
a speed optimisation.
- Utilising shmem MAP_SHARED on the host allows the host to efficiently
share pages.

Clear Containers uses the following steps to set up the DAX mappings:
- QEMU is configured with an nvdimm memory device, with a memory file
backend to map in the host side file into the virtual nvdimm space.
- The guest kernel command line mounts this nvdimm device with the DAX
feature enabled, allowing direct page mapping and access, thus bypassing the
guest page cache.

![DAX](https://raw.githubusercontent.com/01org/cc-oci-runtime/master/documentation/DAX.png)

More information about DAX can be found in the Linux Kernel
[documentation](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/filesystems/dax.txt)

Information on the use of nvdimm via QEMU is available in the QEMU source code
[here](http://git.qemu-project.org/?p=qemu.git;a=blob;f=docs/nvdimm.txt;hb=HEAD)

