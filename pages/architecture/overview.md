---
title: Architecture Overview
sidebar: architecture_sidebar
permalink: arch-overview.html
folder: architecture
---

This is an architectural overview of Clear Containers, based on the 2.1 release.

The [Clear Containers runtime (cc-oci-runtime)](https://github.com/01org/cc-oci-runtime)
complies with the [OCI](https://github.com/opencontainers) specifications and thus
works seamlessly with the [Docker Engine](https://www.docker.com/products/docker-engine)
pluggable runtime architecture. In other words, one can transparently replace the
[default Docker runtime (runc)](https://github.com/opencontainers/runc) with `cc-oci-runtime`.

![Docker and Clear Containers](https://raw.githubusercontent.com/01org/cc-oci-runtime/master/documentation/docker-cc.png)

`cc-oci-runtime` creates a QEMU/KVM virtual machine for each container the Docker engine creates.

The container process is then spawned by [hyperstart](https://github.com/hyperhq/hyperstart/),
an agent running as a daemon on the guest operating system.
Hyperstart opens 2 virtio serial interfaces (Control and I/O) on the guest, and QEMU exposes them
as serial devices on the host. `cc-oci-runtime` uses the control device for sending container
management commands to hyperstart while the I/O serial device is used to pass I/O streams (`stdout`,
`stderr`, `stdin`) between the guest and the Docker Engine.

For any given container, both the init process and all potentially executed commands within that
container, together with their related I/O streams, need to go through 2 virtio serial interfaces
exported by QEMU. The [Clear Containers proxy (`cc-proxy`)](https://github.com/01org/cc-oci-runtime/tree/master/proxy)
multiplexes and demultiplexes those commands and streams for all container virtual machines.
There is only one `cc-proxy` instance running per Clear Containers host.

On the host, each container process is reaped by a Docker specific (`containerd-shim`) monitoring
daemon. As Clear Containers processes run inside their own virtual machines, `containerd-shim`
can not monitor, control or reap them. `cc-oci-runtime` fixes that issue by creating an
[additional shim process (`cc-shim`)](https://github.com/01org/cc-oci-runtime/tree/master/shim)
between `containerd-shim` and `cc-proxy`. A `cc-shim` instance will both forward signals and `stdin`
streams to the container process on the guest and pass the container `stdout` and `stderr` streams
back to the Docker engine via `containerd-shim`.
`cc-oci-runtime` creates a `cc-shim` daemon for each Docker container and for each command Docker
wants to run within an already running container (`docker exec`).

The container workload, i.e. the actual OCI bundle rootfs, is exported from the host to
the virtual machine via a 9pfs virtio mount point. Hyperstart uses this mount point as the root
filesystem for the container processes.

![Overall architecture](https://raw.githubusercontent.com/01org/cc-oci-runtime/master/documentation//overall-architecture.png)

