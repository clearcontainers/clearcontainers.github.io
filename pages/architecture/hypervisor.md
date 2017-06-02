---
title: Hypervisor
sidebar: architecture_sidebar
permalink: arch-hypervisor.html
folder: architecture
---

Clear Containers use [KVM](http://www.linux-kvm.org/page/Main_Page)/[QEMU](http://www.qemu-project.org/) to
create virtual machines where Docker containers will run:

![QEMU/KVM](https://raw.githubusercontent.com/01org/cc-oci-runtime/master/documentation/qemu.png)

Although Clear Containers can run with any recent QEMU release, containers boot time and memory
footprint are significantly optimized by using a specific QEMU version called [`qemu-lite`](https://github.com/01org/qemu-lite).

`qemu-lite` improvements comes through a new `pc-lite` machine type, mostly by:
- Removing many of the legacy hardware devices support so that the guest kernel does not waste
time initializing devices of no use for containers.
- Skipping the guest BIOS/firmware and jumping straight to the Clear Containers kernel.
