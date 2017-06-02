---
title: Networking
sidebar: architecture_sidebar
permalink: arch-networking.html
folder: architecture
---

Containers will typically live in their own, possibly shared, networking namespace.
At some point in a container lifecycle, container engines will set up that namespace
to add the container to a network which is isolated from the host network, but which is shared between containers

In order to do so, container engines will usually add one end of a `virtual ethernet
(veth)` pair into the container networking namespace. The other end of the `veth` pair
is added to the container network.

This is a very namespace-centric approach as QEMU can not handle `veth` interfaces.
Instead it typically creates `TAP` interfaces for adding connectivity to a virtual
machine.

To overcome that incompatibility between typical container engines expectations
and virtual machines, `cc-oci-runtime` networking transparently bridges `veth`
interfaces with `TAP` ones:

![Clear Containers networking](https://raw.githubusercontent.com/01org/cc-oci-runtime/master/documentation/network.png)

The [virtcontainers library](https://github.com/containers/virtcontainers#cnm) has some more
details on how `cc-oci-runtime` implements [CNM](https://github.com/docker/libnetwork/blob/master/docs/design.md).

