---
layout: user-manual
project: atomix
menu: user-manual
title: Primitive Protocols
---

Distributed primitives are modelled as partitioned abstract replicated state machines. When a primitive is created, it can be mapped to a specific [partition group][partition-groups] and configured with a protocol configuration specific to that group. The protocol configuration defines how the primitive behaves with respect to the protocol implemented in the selected partition group, e.g. with respect to consistency models, communication patterns, timeouts, and retries.

For example, to configure a [`DistributedLock`][DistributedLock] primitive to run on the Raft consensus protocol, a cluster must be first configured with a [`RaftPartitionGroup`][RaftPartitionGroup] and then the lock primitive configured with a [`MultiRaftProtocol`][MultiRaftProtocol] configuration. This instructs Atomix to replicate the lock within the indicated Raft [`PartitionGroup`][PartitionGroup] using the given multi-Raft protocol configuration.

## MultiRaftProtocol

The [`MultiRaftProtocol`][MultiRaftProtocol] is the protocol configuration required by [`RaftPartitionGroup`][RaftPartitionGroup]s. To replicate a primitive using the Raft consensus protocol, the cluster must first be configured with a Raft partition group:

```yaml
cluster:
  local-member-id: member-1
  members:
    member-1:
      type: persistent
      address: localhost:5001
    member-2:
      type: persistent
      address: localhost:5002
    member-3:
      type: persistent
      address: localhost:5003
management-group:
  type: raft
  name: system
  partitions: 1
  members:
    - member-1
    - member-2
    - member-3
partition-groups:
  raft:
    type: raft
    partitions: 7
    members:
      - member-1
      - member-2
      - member-3
```

The groups listed under the `partition-groups` section of the configuration are accessible to distributed primitives. To create a primitive replicated in the Raft partition group named `raft`, construct a `MultiRaftProtocol` configuration indicating the Raft partition group name:

```java
Atomix atomix = new Atomix("my.yaml");
atomix.start().join();

DistributedLock lock = atomix.lockBuilder("my-lock")
  .withProtocol(MultiRaftProtocol.builder("raft")
    .withReadConsistency(ReadConsistency.LINEARIZABLE)
    .withCommunicationStrategy(CommunicationStrategy.LEADER)
    .build())
  .build();
```

This will construct a [`DistributedLock`][DistributedLock] primitive stored in the `raft` Raft partition group in a strongly consistent manner - reading directly from the Raft partition leaders using `LINEARIZABLE` read consistency.

As shorthand, the [`MultiRaftProtocol`][MultiRaftProtocol] instance can be configured without a partition group name if the `raft` partition group is the only `RaftPartitionGroup` configured in the cluster:

```java
DistributedLock lock = atomix.lockBuilder("my-lock")
  .withProtocol(MultiRaftProtocol.builder()
    ...
    .build())
  .build();
```

Individual primitives' protocols may also be configured via the Atomix configuration files. To configure a primitive to use the multi-Raft protocol, use the `multi-raft` protocol type:

```yaml
primitives:
  my-lock:
    protocol:
      type: multi-raft
      read-consistency: linearizable
      communication-strategy: leader
```

## MultiPrimaryProtocol

The multi-primary protocol is used to configure primitives to be replicated in a [`PrimaryBackupPartitionGroup`][PrimaryBackupPartitionGroup]. Multi-primary protocols are designed for high scalability and availability. Users can configure the number of backups to maintain in each partition and whether to backup the primitive synchronously or asynchronously.

To use multi-primary primitives, the cluster must first be configured with a [`PrimaryBackupPartitionGroup`][PrimaryBackupPartitionGroup]:

```yaml
cluster:
  local-member-id: member-1
  members:
    member-1:
      type: persistent
      address: localhost:5001
    member-2:
      type: persistent
      address: localhost:5002
    member-3:
      type: persistent
      address: localhost:5003
management-group:
  type: raft
  name: system
  partitions: 1
  members:
    - member-1
    - member-2
    - member-3
partition-groups:
  data:
    type: primary-backup
    partitions: 32
```

To configure a multi-primary-based primitive, use the [`MultiPrimaryProtocol`][MultiPrimaryProtocol] builder, passing the name of the primary-backup group to the `builder` method:

```java
Atomix atomix = new Atomix("my.yaml");
atomix.start().join();

ConsistentMap<String, String> map = atomix.<String, String>consistentMapBuilder("my-map")
  .withProtocol(MultiPrimaryProtocol.builder("data")
    .withNumBackups(2)
    .withReplication(Replication.ASYNCHRONOUS)
    .build())
  .build();
```

Individual primitives' protocols may also be configured via the Atomix configuration files. To configure a primitive to use the multi-primary protocol, use the `multi-primary` protocol type:

```yaml
primitives:
  my-map:
    protocol:
      type: multi-primary
      backups: 2
      replication: asynchronous
```

## Protocol Partitioners

Many distributed primitives are partitioned among all the partitions in the configured [`PartitionGroup`][PartitionGroup]. For example, when putting a key/value in a [`ConsistentMap`][ConsistentMap], the key is mapped to a partition using a configured [`Partitioner`][Partitioner]. This allows the cluster to scale by spreading data across multiple partitions.

For partitioned primitives, most primitive implementations encode keys to strings and then use the default Murmur 3 hash to map the key to a partition. Users can provide custom [`Partitioner`][Partitioner]s to alter this behavior in the protocol configuration:

```java
ConsistentMap<String, String> map = atomix.<String, String>consistentMapBuilder("my-map")
  .withProtocol(MultiPrimaryProtocol.builder()
    .withPartitioner((key, partitions) -> partitions.get(Math.abs(key.hashCode() % partitions.size())))
    .withNumBackups(2)
    .build())
  .build();
```

{% include common-links.html %}
