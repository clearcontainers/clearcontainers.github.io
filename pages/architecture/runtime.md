---
title: Runtime
sidebar: architecture_sidebar
permalink: arch-runtime.html
folder: architecture
---

`cc-oci-runtime` is an OCI compatible container runtime and is responsible for handling all
commands specified by [the OCI runtime specification](https://github.com/opencontainers/runtime-spec)
and launching `cc-shim` instances.

Here we will describe how `cc-oci-runtime` handles the most important OCI commands.

### `create`

When handling the OCI `create` command, `cc-oci-runtime` goes through the following steps:

1. Create the container namespaces (Only the network and mount namespaces are currently supported),
according to the container OCI configuration file.
2. Spawn the `cc-shim` process and have it wait on a couple of temporary pipes for:
   * A `cc-proxy` created file descriptor (one end of a socketpair) for the shim to connect to.
   * The container `hyperstart` sequence numbers for at most 2 I/O streams (One for `stdout` and `stdin`, one for `stderr`).
   `hyperstart` uses those sequence numbers to multiplex all streams for all processes through one serial interface (The
   virtio I/O serial one).
3. Run all the [OCI hooks](https://github.com/opencontainers/runtime-spec/blob/master/config.md#hooks) in the container namespaces,
as described by the OCI container configuration file.
4. [Set up the container networking](https://github.com/01org/cc-oci-runtime/blob/master/documentation/architecture.md#networking).
This must happen after all hooks are done as one of them is potentially setting
the container networking namespace up.
5. Create the virtual machine running the container process. The VM `systemd` instance will spawn the `hyperstart` daemon.
6. Wait for `hyperstart` to signal that it is ready.
7. Send the pod creation command to `hyperstart`. The `hyperstart` pod is the container process sandbox.
8. Send the allocateIO command to the proxy, for getting the `hyperstart` I/O sequence numbers described in step 2.
9. Pass the `cc-proxy` socketpair file descriptor, and the I/O sequence numbers to the listening cc-shim process through the dedicated pipes.
10. The `cc-shim` instance is put into a stopped state to prevent it from doing I/O before the container is started.

![Docker create](https://raw.githubusercontent.com/01org/cc-oci-runtime/master/documentation/create.png)

At that point, the container sandbox is created in the virtual machine and `cc-shim` is stopped on the host.
However the container process itself is not yet running as one needs to call `docker start` to actually start it.

### `start`

On namespace containers `start` launches a traditional Linux container process in its own set of namespaces.
With Clear Containers, the main task of `cc-oci-runtime` is to create and start a container within the
pod that got created during the `create` step. In practice, this means `cc-oci-runtime` follows
these steps:

1. `cc-oci-runtime` connects to `cc-proxy` and sends it the `attach` command to let it know which pod
we want to use to create and start the new container.
2. `cc-oci-runtime` sends a hyperstart `NEWCONTAINER` command to create and start a new container in
a given pod. The command is sent to `cc-proxy` who forwards it to the right hyperstart instance running
in the appropriate guest.
3. `cc-oci-runtime` resumes `cc-shim` so that it can now connect to the `cc-proxy` and acts as
a signal and I/O streams proxy between `containerd-shim` and `cc-proxy`.

### `exec`

`docker exec` allows one to run an additional command within an already running container.
With Clear Containers, this translates into sending a `EXECCMD` command to hyperstart so
that it runs a command into a running container belonging to a certain pod.
All I/O streams from the executed command will be passed back to Docker through a newly
created `cc-shim`.

The `exec` code path is partly similar to the `create` one and `cc-oci-runtime` goes through
the following steps:

1. `cc-oci-runtime` connects to `cc-proxy` and sends it the `attach` command to let it know which pod
we want to use to run the `exec` command.
2. `cc-oci-runtime` sends the allocateIO command to the proxy, for getting the `hyperstart` I/O sequence
numbers for the `exec` command I/O streams.
3. `cc-oci-runtime` sends an hyperstart `EXECMD` command to start the command in the right container
The command is sent to `cc-proxy` who forwards it to the right hyperstart instance running
in the appropriate guest.
4. Spawn the `cc-shim` process for it to forward the output streams (stderr and stdout) and the `exec`
command exit code to Docker.

Now the `exec`'ed process is running in the virtual machine, sharing the UTS, PID, mount and IPC
namespaces with the container's init process.

### `kill`

When sending the OCI `kill` command, container runtimes should send a [UNIX signal](https://en.wikipedia.org/wiki/Unix_signal)
to the container process.
In the Clear Containers context, this means `cc-oci-runtime` needs a way to send a signal
to the container process within the virtual machine. As `cc-shim` is responsible for
forwarding signals to its associated running containers, `cc-oci-runtime` naturally
calls `kill` on the `cc-shim` PID.

However, `cc-shim` is not able to trap `SIGKILL` and `SIGSTOP` and thus `cc-oci-runtime`
needs to follow a different code path for those 2 signals.
Instead of `kill`'ing the `cc-shim` PID, it will go through the following steps:

1. `cc-oci-runtime` connects to `cc-proxy` and sends it the `attach` command to let it know
on which pod the container it is trying to `kill` is running.
2. `cc-oci-runtime` sends an hyperstart `KILLCONTAINER` command to `kill` the container running
on the guest. The command is sent to `cc-proxy` who forwards it to the right hyperstart instance
running in the appropriate guest.

### `delete`

`docker delete` is about deleting all resources held by a stopped/killed container. Running
containers can not be `delete`d unless the OCI runtime is explictly being asked to. In that
case it will first `kill` the container and only then `delete` it.

The resources held by a Clear Container are quite different from the ones held by a host
namespace container e.g. run by `runc`. `cc-oci-runtime` needs mostly to delete the pod
holding the stopped container on the virtual machine, shut the hypervisor down and finally
delete all related proxy resources:

1. `cc-oci-runtime` connects to `cc-proxy` and sends it the `attach` command to let it know
on which pod the container it is trying to to `delete` is running.
2. `cc-oci-runtime` sends an hyperstart `DESTROYPOD` command to `destroy` the pod holding the
container running on the guest. The command is sent to `cc-proxy` who forwards it to the right
hyperstart instance running in the appropriate guest.
3. After deleting the last running pod, `hyperstart` will gracefully shut the virtual machine
down.
4. `cc-oci-runtime` sends the `BYE` command to `cc-proxy`, to let it know that a given virtual
machine is shut down. `cc-proxy` will then clean all its internal resources associated with this
VM.

