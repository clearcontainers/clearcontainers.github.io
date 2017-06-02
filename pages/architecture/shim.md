---
title: Shim
sidebar: architecture_sidebar
permalink: arch-shim.html
folder: architecture
---

Docker's `containerd-shim` is designed around the assumption that it can monitor and reap the actual
container process. As `containerd-shim` runs on the host, it can not directly monitor a process running
within a virtual machine. At most it can see the QEMU process, but that is not enough.
With Clear Containers, `cc-shim` acts as the container process that `containerd-shim` can monitor. Therefore
`cc-shim` needs to handle all container I/O streams (`stdout`, `stdin` and `stderr`) and forward all signals
`containerd-shim` decides to send to the container process.

`cc-shim` has an implicit knowledge about which VM agent will handle those streams and signals and thus act as
an encapsulation layer between `containerd-shim` and `hyperstart`:

- It fragments and encapsulates the standard input stream from containerd-shim into `hyperstart` stream packets:
```
  ┌───────────────────────────┬───────────────┬────────────────────┐
  │ IO stream sequence number │ Packet length │ IO stream fragment │
  │         (8 bytes)         │    (4 bytes)  │                    │
  └───────────────────────────┴───────────────┴────────────────────┘
```
- It de-encapsulates and assembles standard output and error `hyperstart` stream packets into an output stream
that it forwards to `containerd-shim`
- It translates all UNIX signals (except `SIGKILL` and `SIGSTOP`) into `hyperstart` `KILLCONTAINER` commands
that it sends to the VM via `cc-proxy` UNIX named socket.

The IO stream sequence numbers are passed from `cc-runtime` to `cc-shim` when the former spawns the latter.
They are generated by `hyperstart` and `cc-oci-runtime` fetches them by sending the `AllocateIO` command to
`cc-proxy`.

As an example, let's say that running the `pwd` command from a container standard input will generate
`/tmp` from the container standard output. `hyperstart` assigned this specific process 8888 and 8889 respectively
as the stdin, stdout and stderr sequence numbers.
With `cc-shim` and Clear Containers, this example would look like:

![cc-shim](https://raw.githubusercontent.com/01org/cc-oci-runtime/master/documentation/shim.png)