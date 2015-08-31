---
layout: content
menu: user-manual
title: Raft consensus algorithm
---

# Raft consensus algorithm

Copycat is built on a standalone, feature-complete implementation of the [Raft consensus algorithm][Raft]. The Raft implementation consists of three Maven submodules:

### copycat-protocol

The `copycat-protocol` submodule provides base interfaces and classes that are shared between both the [client](#copycat-client) and [server](#copycat-server) modules. The most notable components of the protocol submodule are [commands][Command] and [queries][Query] with which the client communicates state machine operations, and [sessions][Session] through which clients and servers communicate.

### copycat-server

The `copycat-server` submodule is a standalone [Raft][Raft] server implementation. The server provides a feature-complete implementation of the [Raft consensus algorithm][Raft], including dynamic cluster membership changes and log compaction.

The primary interface to the `copycat-server` module is [RaftServer][RaftServer].

### copycat-client

The `copycat-client` submodule provides a [RaftClient][RaftClient] interface for submitting [commands][Command] and [queries][Query] to a cluster of [RaftServer][RaftServer]s. The client implementation includes full support for linearizable commands via [sessions][Session].

## RaftClient

The [RaftClient][RaftClient] provides an interface for submitting [commands](#commands) and [queries](#queries) to a cluster of [Raft servers](#raftserver).

To create a client, you must supply the client [Builder](#builders) with a set of `Members` to which to connect.

```java
Members members = Members.builder()
  .addMember(new Member(1, "123.456.789.1" 5555))
  .addMember(new Member(2, "123.456.789.2" 5555))
  .addMember(new Member(3, "123.456.789.3" 5555))
  .build();
```

The provided `Members` do not have to be representative of the full Copycat cluster, but they do have to provide at least one correct server to which the client can connect. In other words, the client must be able to communicate with at least one `RaftServer` that is the leader or can communicate with the leader, and a majority of the cluster must be able to communicate with one another in order for the client to register a new [Session](#session).

```java
RaftClient client = RaftClient.builder()
  .withTransport(new NettyTransport())
  .withMembers(members)
  .build();
```

Once a `RaftClient` has been created, connect to the cluster by calling `open()` on the client:

```java
client.open().thenRun(() -> System.out.println("Successfully connected to the cluster!"));
```

### Client lifecycle

When the client is opened, it will connect to a random server and attempt to register its session. If session registration fails, the client will continue to attempt registration via random servers until all servers have been tried. If the session cannot be registered, the `CompletableFuture` returned by `open()` will fail.

### Client sessions

Once the client's session has been registered, the `Session` object can be accessed via `RaftClient.session()`.

The client will remain connected to the server through which the session was registered for as long as possible. If the server fails, the client can reconnect to another random server and maintain its open session.

The `Session` object can be used to receive events `publish`ed by the server's `StateMachine`. To register a session event listener, use the `onReceive` method:

```java
client.session().onReceive(message -> System.out.println("Received " + message));
```

When events are sent from a server state machine to a client via the `Session` object, only the server to which the client is connected will send the event. Copycat servers guarantee that state machine events will be received by the client session in the order in which they're sent even if the client switches servers.

## RaftServer

The [RaftServer][RaftServer] class is a feature complete implementation of the [Raft consensus algorithm][Raft]. `RaftServer` underlies all distributed resources supports by Copycat's high-level APIs.

The `RaftServer` class is provided in the `copycat-server` module:

```
<dependency>
  <groupId>net.kuujo.copycat</groupId>
  <artifactId>copycat-server</artifactId>
  <version>{{ site.version }}</version>
</dependency>
```

Each `RaftServer` consists of three essential components:

* [Transport](#transports) - Used to communicate with clients and other Raft servers
* [Storage](#storage) - Used to persist [commands](#commands) to memory or disk
* [StateMachine](#state-machines) - Represents state resulting from [commands](#commands) logged and replicated via Raft

To create a Raft server, use the server [Builder](#builders):

```java
RaftServer server = RaftServer.builder()
  .withMemberId(1)
  .withMembers(members)
  .withTransport(new NettyTransport())
  .withStorage(Storage.builder()
    .withStorageLevel(StorageLevel.MEMORY)
    .build())
  .withStateMachine(new MyStateMachine())
  .build();
```

Once the server has been created, call `open()` to start the server:

```java
server.open().thenRun(() -> System.out.println("Server started successfully!"));
```

The returned `CompletableFuture` will be completed once the server has connected to other members of the cluster and, critically, discovered the cluster leader. See the [server lifecycle](#server-lifecycle) for more information on how the server joins the cluster.

### Server lifecycle

Copycat's Raft implementation supports dynamic membership changes designed to allow servers to arbitrarily join and leave the cluster. When a `RaftServer` is configured, the `Members` list provided in the server configuration specifies some number of servers to join to form a cluster. When the server is started, the server begins a series of steps to either join an existing Raft cluster or start a new cluster:

* When the server starts, transition to a *join* state and attempt to join the cluster by sending a *join* request to each known `Member` of the cluster
* If, after an election timeout, the server has failed to receive a response to a *join* requests from any `Member` of the cluster, assume that the cluster doesn't exist and transition into the *follower* state
* Once a leader has been elected or otherwise discovered, complete the startup

When a member *joins* the cluster, a *join* request will ultimately be received by the cluster's leader. The leader will log and replicate the joining member's configuration. Once the joined member's configuration has been persisted on a majority of the cluster, the joining member will be notified of the membership change and transition to the *passive* state. While in the *passive* state, the joining member cannot participate in votes but does receive *append* requests from the cluster leader. Once the leader has determined that the joining member's log has caught up to its own (the joining node's log has the last committed entry at any given point in time), the member is promoted to a full member via another replicated configuration change.

Once a node has fully joined the Raft cluster, in the event of a failure the quorum size will not change. To leave the cluster, the `close()` method must be called on a `RaftServer` instance. When `close()` is called, the member will submit a *leave* request to the leader. Once the leaving member's configuration has been removed from the cluster and the new configuration replicated and committed, the server will complete the close.

## Commands

Commands are operations that modify the state machine state. When a command operation is submitted to the Copycat cluster, the command is logged to disk or memory (depending on the [Storage](#storage) configuration) and replicated via the Raft consensus protocol. Once the command has been stored on a majority cluster members, it will be applied to the server-side [StateMachine](#state-machines) and the output will be returned to the client.

Commands are defined by implementing the `Command` interface:

```java
public class Set<T> implements Command<T> {
  private final String value;

  public Set(String value) {
    this.value = value;
  }

  /**
   * The value to set.
   */
  public String value() {
    return value;
  }
}
```

The [Command][Command] interface extends [Operation][Operation] which is `Serializable` and can be sent over the wire with no additional configuration. However, for the best performance users should implement [CopycatSerializable][CopycatSerializable] or register a [TypeSerializer][TypeSerializer] for the type. This will reduce the size of the serialized object and allow Copycat's [Serializer](#serializer) to optimize class loading internally during deserialization.

## Queries

In contrast to commands which perform state change operations, queries are read-only operations which do not modify the server-side state machine's state. Because read operations do not modify the state machine state, Copycat can optimize queries according to read from certain nodes according to the configuration and [may not require contacting a majority of the cluster in order to maintain consistency](#query-consistency). This means queries can significantly reduce disk and network I/O depending on the query configuration, so it is strongly recommended that all read-only operations be implemented as queries.

To create a query, simply implement the [Query][Query] interface:

```java
public class Get<T> implements Query {
}
```

As with [Command][Command], [Query][Query] extends the base [Operation][Operation] interface which is `Serializable`. However, for the best performance users should implement [CopycatSerializable][CopycatSerializable] or register a [TypeSerializer][TypeSerializer] for the type.

### Query consistency

By default, [queries](#queries) submitted to the Copycat cluster are guaranteed to be linearizable. Linearizable queries are forwarded to the leader where the leader verifies its leadership with a majority of the cluster before responding to the request. However, this pattern can be inefficient for applications with less strict read consistency requirements. In those cases, Copycat allows [Query][Query] implementations to specify a `ConsistencyLevel` to control how queries are handled by the cluster.

To configure the consistency level for a `Query`, simply override the default `consistency()` getter:

```java
public class Get<T> implements Query {

  @Override
  public ConsistencyLevel consistency() {
    return Consistency.SERIALIZABLE;
  }

}
```

The consistency level returned by the overridden `consistency()` method amounts to a *minimum consistency requirement*. In many cases, a `SERIALIZABLE` query can actually result in `LINEARIZABLE` read depending the server to which a client submits queries, but clients can only rely on the configured consistency level.

Copycat provides four consistency levels:
* `ConsistencyLevel.LINEARIZABLE` - Provides guaranteed linearizability by forcing all reads to go through the leader and verifying leadership with a majority of the Raft cluster prior to the completion of all operations
* `ConsistencyLevel.LINEARIZABLE_LEASE` - Provides best-effort optimized linearizability by forcing all reads to go through the leader but allowing most queries to be executed without contacting a majority of the cluster so long as less than the election timeout has passed since the last time the leader communicated with a majority
* `ConsistencyLevel.SERIALIZABLE` - Provides serializable consistency by allowing clients to read from followers and ensuring that clients see state progress monotonically

## State machines

State machines are the server-side representation of state based on a series of [commands](#commands) and [queries](#queries) submitted to the Raft cluster.

**All state machines must be deterministic**

Given the same commands in the same order, state machines must always arrive at the same state with the same output.

Non-deterministic state machines will break the guarantees of the Raft consensus algorithm. Each [server](#raftserver) in the cluster must have *the same state machine*. When a command is submitted to the cluster, the command will be forwarded to the leader, logged to disk or memory, and replicated to a majority of the cluster before being applied to the state machine, and the return value for a given command or query is returned to the requesting client.

State machines are created by extending the base `StateMachine` class and overriding the `configure(StateMachineExecutor)` method:

```java
public class MyStateMachine extends StateMachine {

  @Override
  protected void configure(StateMachineExecutor executor) {
  
  }

}
```

Internally, state machines are backed by a series of entries in an underlying [log](#log). In the event of a crash and recovery, state machine commands in the log will be replayed to the state machine. This is why it's so critical that state machines be deterministic.

#### Registering operations

The `StateMachineExecutor` is a special [Context](#contexts) implemntation that is responsible for applying [commands](#commands) and [queries](#queries) to the state machine. Operations are handled by registering callbacks on the provided `StateMachineExecutor` in the `configure` method:

```java
@Override
protected void configure(StateMachineExecutor executor) {
  executor.register(SetCommand.class, this::set);
  executor.register(GetQuery.class, this::get);
}
```

### Commits

As [commands](#commands) and [queries](#queries) are logged and replicated through the Raft cluster, they gain some metadata that is not present in the original operation. By the time operations are applied to the state machine, they've gained valuable information that is exposed in the [Commit][Commit] wrapper class:

* `Commit.index()` - The sequential index of the commit in the underlying `Log`. The index is guaranteed to increase monotonically as commands are applied to the state machine. However, because [queries](#queries) are not logged, they may duplicate the indices of commands.
* `Commit.time()` - The approximate `Instant` at which the commit was logged by the leader through which it was committed. The commit time is guaranteed never to decrease.
* `Commit.session()` - The [Session](#sessions) that submitted the operation to the cluster. This can be used to send events back to the client.
* `Commit.operation()` - The operation that was committed.

```java
protected Object get(Commit<GetQuery> commit) {
  return map.get(commit.operation().key());
}
```

### Sessions

Sessions are representative of a single client's connection to the cluster. For each `Commit` applied to the state machine, an associated `Session` is provided. State machines can use sessions to associate clients with state changes or even send events back to the client through the session:

```java
protected Object put(Commit<PutCommand> commit) {
  commit.session().publish("putteded");
  return map.put(commit.operation().key(), commit.operation().value());
}
```

The `StateMachineContext` provides a view of the local server's state at the time a [command](#command) or [query](#queries) is applied to the state machine. Users can use the context to access, for instance, the list of `Session`s currently registered in the cluster.

To get the context, call the protected `context()` getter from inside the state machine:

```java
for (Session session : context().sessions()) {
  session.publish("Hello world!");
}
```

### Commit cleaning

As commands are submitted to the cluster and applied to the Raft state machine, the underlying [log](#log) grows. Without some mechanism to reduce the size of the log, the log would grow without bound and ultimately servers would run out of disk space. Raft suggests a few different approaches of handling log compaction. Copycat uses the [log cleaning](#log-cleaning) approach.

`Commit` objects are backed by entries in Copycat's replicated log. When a `Commit` is no longer needed by the `StateMachine`, the state machine should clean the commit from Copycat's log by calling the `clean()` method:

```java
protected void remove(Commit<RemoveCommand> commit) {
  map.remove(commit.operation().key());
  commit.clean();
}
```

Internally, the `clean()` call will be proxied to Copycat's underlying log:

```java
log.clean(commit.index());
```

As commits are cleaned by the state machine, entries in the underlying log will be marked for deletion. *Note that it is not safe to assume that once a commit is cleaned it is permanently removed from the log*. Cleaning an entry only *marks* it for deletion, and the entry won't actually be removed from the log until a background thread cleans the relevant log segment. This means in the event of a crash-recovery and replay of the log, a previously `clean`ed commit may still exists. For this reason, if a commit is dependent on a prior commit, state machines should only `clean` those commits if no prior related commits have been seen. (More on this later)

Once the underlying `Log` has grown large enough, and once enough commits have been `clean`ed from the log, a pool of background threads will carry out their task to rewrite segments of the log to remove commits (entries) for which `clean()` has been called:

#### Deterministic scheduling

In addition to registering operation callbacks, the `StateMachineExecutor` also facilitates deterministic scheduling based on the Raft replicated log.

```java
executor.schedule(() -> System.out.println("Every second"), Duration.ofSeconds(1), Duration.ofSeconds(1));
```

Because of the complexities of coordinating distributed systems, time does not advance at the same rate on all servers in the cluster. What is essential, though, is that time-based callbacks be executed at the same point in the Raft log on all nodes. In order to accomplish this, the leader writes an approximate `Instant` to the replicated log for each command. When a command is applied to the state machine, the command's timestamp is used to invoke any outstanding scheduled callbacks. This means the granularity of scheduled callbacks is limited by the minimum time between commands submitted to the cluster, including session register and keep-alive requests. Thus, users should not rely on `StateMachineExecutor` scheduling for accuracy.

[Javadoc]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/
[CAP]: https://en.wikipedia.org/wiki/CAP_theorem
[Raft]: https://raft.github.io/
[Executor]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html
[CompletableFuture]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html
[collections]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/collections.html
[atomic]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/atomic.html
[coordination]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/coordination.html
[copycat]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat.html
[protocol]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/raft/protocol.html
[io]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io.html
[serializer]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/serializer.html
[transport]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/transport.html
[storage]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/storage.html
[utilities]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/util.html
[Copycat]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/Copycat.html
[CopycatReplica]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/CopycatReplica.html
[CopycatClient]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/CopycatClient.html
[Resource]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/Resource.html
[Transport]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/transport/Transport.html
[LocalTransport]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/transport/LocalTransport.html
[NettyTransport]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/transport/NettyTransport.html
[Storage]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/storage/Storage.html
[Log]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/storage/Log.html
[Buffer]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/Buffer.html
[BufferReader]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/BufferReader.html
[BufferWriter]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/BufferWriter.html
[Serializer]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/serializer/Serializer.html
[CopycatSerializable]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/serializer/CopycatSerializable.html
[TypeSerializer]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/serializer/TypeSerializer.html
[SerializableTypeResolver]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/serializer/SerializableTypeResolver.html
[PrimitiveTypeResolver]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/serializer/SerializableTypeResolver.html
[JdkTypeResolver]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/serializer/SerializableTypeResolver.html
[ServiceLoaderTypeResolver]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/io/serializer/ServiceLoaderTypeResolver.html
[RaftServer]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/raft/RaftServer.html
[RaftClient]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/raft/RaftClient.html
[Session]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/raft/session/Session.html
[Operation]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/raft/protocol/Operation.html
[Command]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/raft/protocol/Command.html
[Query]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/raft/protocol/Query.html
[Commit]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/raft/protocol/Commit.html
[ConsistencyLevel]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/raft/protocol/ConsistencyLevel.html
[DistributedAtomicValue]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/atomic/DistributedAtomicValue.html
[DistributedSet]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/collections/DistributedSet.html
[DistributedMap]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/collections/DistributedMap.html
[DistributedLock]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/coordination/DistributedLock.html
[DistributedLeaderElection]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/coordination/DistributedLeaderElection.html
[DistributedTopic]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/coordination/DistributedTopic.html
[Builder]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/util/Builder.html
[Listener]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/util/Listener.html
[Context]: http://kuujo.github.io/copycat/api/{{ site.javadoc-version }}/net/kuujo/copycat/util/concurrent/Context.html