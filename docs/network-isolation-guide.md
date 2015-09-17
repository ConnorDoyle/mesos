---
layout: documentation
---

# Framework writers&apos; guide to IP-per-container network isolation

Mesos 0.25.0 introduced primitives for network isolation based on
assigning one or more IP addresses to containers.

This document covers mechanics and interaction points from the
perspective of the scheduler.  For a comprehensive view of the network
isolation primitives see the
[network isolation documentation](network-isolation.md).

## Requesting one or more IP addresses

> Custom isolation, like containerization, is a per-executor concept due
> to the flexible task execution model that Mesos provides.  This means
> that it is not currently possible to express unique IP addresses for
> two tasks for the same executor.

To maximize backwards-compatibility with existing frameworks, schedulers
must opt-in to network isolation per-container.  Schedulers opt in to
network isolation using new data structures in the `TaskInfo` message.

There is a message called `NetworkInfo`:

~~~{.proto}
message NetworkInfo {
  enum Protocol {
    IPv4 = 1;
    IPv6 = 2;
  }
  optional Protocol protocol = 1;
  optional string ip_address = 2;
  repeated string groups = 3;
  optional Labels labels = 4;
};
~~~

The `NetworkInfo` message can be added to `ContainerInfo`.

~~~{.proto}
message ContainerInfo {
  ...
  repeated NetworkInfo network_infos;
  ...
};
~~~

> The Mesos containerizer will reject any `TaskInfo` that directly has
> a ContainerInfo.  For this reason, when opting in to network isolation
> when using the Mesos containerizer, set
> `TaskInfo.ExecutorInfo.ContainerInfo.NetworkInfo` instead.

## Address discovery

The `NetworkInfo` message allows schedulers to request IP addres(es) to be
assigned at task launch time on the Mesos agent.  After opting in to
network isolation for a given executor’s container in this way, many
schedulers will need to know what address(es) were ultimately assigned
in order to perform health checks, or any other out-of-band
communication.

This is accomplished via a field in the `TaskStatus` message.

~~~{.proto}
message ContainerStatus {
  repeated NetworkInfo network_infos;
}
~~~

~~~{.proto}
message TaskStatus {
  ...
  optional ContainerStatus container;
  ...
};
~~~

As of Mesos version 0.25.0 the `ContainerStatus` message is the standard way
to discover the reachable address(es) for tasks.  In other words, it is no
longer generally valid to assume that a task is reachable on the network using
the agent hostname from the resource offer.
