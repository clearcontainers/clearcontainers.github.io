---
title: Hyperstart
sidebar: architecture_sidebar
permalink: arch-hyperstart.html
folder: architecture
---

[`Hyperstart`](https://github.com/hyperhq/hyperstart) is a daemon running on the guest as an
agent for managing containers and processes potentially running within those containers.

It is statically built out of a compact C code base, with a strong focus on both simplicity
and memory footprint.
The `hyperstart` execution unit is the pod. A `hyperstart` pod is a container sandbox defined
by a set of namespaces (UTS, PID, mount and IPC). Although a pod can hold several containers,
`cc-oci-runtime` always runs a single container per pod.

`Hyperstart` sends and receives [specific commands](https://github.com/hyperhq/hyperstart/blob/master/src/api.h)
over a control serial interface for controlling and managing pods and containers. For example,
`cc-oci-runtime` will send the following `hyperstart` commands sequence when starting a container:

* `STARTPOD` creates a Pod sandbox and takes a `Pod` structure as its argument:
```Go
type Pod struct {
	Hostname              string                `json:"hostname"`
	DeprecatedContainers  []Container           `json:"containers,omitempty"`
	DeprecatedInterfaces  []NetworkInf          `json:"interfaces,omitempty"`
	Dns                   []string              `json:"dns,omitempty"`
	DeprecatedRoutes      []Route               `json:"routes,omitempty"`
	ShareDir              string                `json:"shareDir"`
	PortmappingWhiteLists *PortmappingWhiteList `json:"portmappingWhiteLists,omitempty"`
}
```
* `NEWCONTAINER` will create and start a container within the previously created
pod. This command takes a container description as its argument:

```Go
type Container struct {
	Id            string              `json:"id"`
	Rootfs        string              `json:"rootfs"`
	Fstype        string              `json:"fstype,omitempty"`
	Image         string              `json:"image"`
	Addr          string              `json:"addr,omitempty"`
	Volumes       []*VolumeDescriptor `json:"volumes,omitempty"`
	Fsmap         []*FsmapDescriptor  `json:"fsmap,omitempty"`
	Sysctl        map[string]string   `json:"sysctl,omitempty"`
	Process       *Process            `json:"process"`
	RestartPolicy string              `json:"restartPolicy"`
	Initialize    bool                `json:"initialize"`
	Ports         []Port              `json:"ports,omitempty"` //deprecated
}
```

`Hyperstart` uses a separate serial channel for passing the container processes output streams
(`stdout`, `stderr`) back to `cc-proxy` and receiving the input stream (`stdin`) for them.
As all streams for all containers are going through one single serial channel, hyperstart
prepends them with container specific sequence numbers. There are at most 2 sequence numbers
per container process, one for `stdout` and `stdin`, and another one for `stderr`.

