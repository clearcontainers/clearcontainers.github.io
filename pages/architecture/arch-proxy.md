---
title: Proxy
sidebar: architecture_sidebar
permalink: arch-proxy.html
folder: architecture
---

`cc-proxy` is a daemon offering access to the [`hyperstart`](https://github.com/hyperhq/hyperstart)
VM agent to multiple `cc-shim` and `cc-oci-runtime` clients.
Only a single instance of `cc-proxy` per host is necessary as it can be used for several different VMs.
Its main role is to:
- Arbitrate access to the `hyperstart` control channel between all the `cc-oci-runtime` instances and the `cc-shim` ones.
- Route the I/O streams between the various `cc-shim` instances and `hyperstart`.

`cc-proxy` provides 2 client interfaces:

- A UNIX, named socket for all `cc-oci-runtime` instances on the host to send commands to `cc-proxy`.
- One socket pair per `cc-shim` instance, to send stdin and receive stdout and stderr I/O streams. See the
[cc-shim section](https://github.com/01org/cc-oci-runtime/blob/master/documentation/architecture.md#shim)
for more details about that interface.

The protocol on the `cc-proxy` UNIX named socket supports the following commands:

- `Hello`: This command is for `cc-oci-runtime` to let `cc-proxy` know about a newly created VM that will
hold containers. This command payload contains the `hyperstart` control and I/O UNIX socket paths created
and exported by QEMU, and `cc-proxy` will connect to both of them after receiving the `Hello` command.
- `Bye`: This is the opposite of `Hello`, i.e. `cc-oci-runtime` uses this command to let `cc-proxy` know
that it can release all resources related to the VM described in the command payload.
- `Attach`: `cc-oci-runtime` uses that command as a VM multiplexer as it allows it to notify `cc-proxy` about
which VM it wants to talk to. In other words, this commands allows `cc-oci-runtime` to attach itself to a
running VM.
- `AllocateIO`: As `hyperstart` can potentially handle I/O streams from multiple container processes at the
same time, it needs to be able to associate any given stream to a container process. This is done by `hyperstart`
allocating a set of at most 2 so-called sequence numbers per container process. `cc-oci-runtime` will send
the `AllocateIO` command to `cc-proxy` to have it request `hyperstart` to allocate those sequence numbers.
They will be passed as command line arguments to `cc-shim`, who will then use them to e.g. prepend its stdin
stream packets with the right sequence number.
- `Hyper`: This command is used by both `cc-oci-runtime` and `cc-shim` to forward `hyperstart` specific
commands.

For more details about `cc-proxy`'s protocol, theory of operations or debugging tips, please read
[`cc-proxy` README](https://github.com/01org/cc-oci-runtime/tree/master/proxy).
