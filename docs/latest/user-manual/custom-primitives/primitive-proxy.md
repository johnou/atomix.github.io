---
layout: user-manual
project: atomix
menu: user-manual
title: Creating a Proxy
---

The primitive service is the stateful component of distributed primitives. Atomix creates multiple instances of a service on different nodes and replicates changes between them. On the client side, primitives must provide a primitive proxy which interacts with the replicated services using service and client proxies.

The client-side primitive proxy should extend the [`AbstractAsyncPrimitiveProxy`][AbstractAsyncPrimitiveProxy] base class, and just as with the primitive service, the client should implement the client proxy interface for the primitive:

```java
public class DistributedLockProxy
    extends AbstractAsyncPrimitiveProxy<AsyncDistributedLock, DistributedLockService>
    implements AsyncDistributedLock, DistributedLockClient {
  private volatile CompletableFuture<Optional<Long>> lockFuture;

  public DistributedLockProxy(PrimitiveProxy proxy, PrimitiveRegistry registry, ScheduledExecutorService scheduledExecutor) {
    super(DistributedLockService.class, proxy, registry);
  }
  
  @Override
  public CompletableFuture<Long> lock() {
    lockFuture = new CompletableFuture<>();
    acceptBy(getPartitionKey(), service -> service.lock())
      .whenComplete((result, error) -> {
        if (error != null) {
          lockFuture.completeExceptionally(error);
        }
      });
    return lockFuture.thenApply(result -> result.get()).whenComplete((r, e) -> lockFuture = null);
  }
  
  @Override
  public CompletableFuture<Optional<Long>> tryLock(Duration timeout) {
    lockFuture = new CompletableFuture<>();
    acceptBy(getPartitionKey(), service -> service.tryLock(timeout.toMillis()))
      .whenComplete((result, error) -> {
        if (error != null) {
          lockFuture.completeExceptionally(error);
        }
      });
    return lockFuture.thenApply(result -> result.get()).whenComplete((r, e) -> lockFuture = null);
  }
  
  @Override
  public CompletableFuture<Void> unlock() {
    return acceptBy(getPartitionKey(), service -> service.unlock());
  }
  
  @Override
  public void locked(long index) {
    CompletableFuture<Optional<Long>> lockFuture = this.lockFuture;
    if (lockFuture != null) {
      lockFuture.complete(Optional.of(index));
    }
  }
  
  @Override
  public void failed() {
    CompletableFuture<Optional<Long>> lockFuture = this.lockFuture;
    if (lockFuture != null) {
      lockFuture.complete(Optional.empty());
    }
  }
}
```

Operations are performed on the primitive service via the service proxy. Service proxy methods are invoked by using one of the following methods:
* `invokeAll`/`acceptAll` - invokes the given callback on all the partitions
```java
invokeAll(service -> service.size())
    .thenApply(results -> results.reduce(Math::addExact).orElse(0));
```
* `invokeOn`/`acceptOn` - invokes the given callback on the given partition
```java
invokeOn(getPartition(1).partitionId(), service -> service.size());
```
* `invokeBy`/`acceptBy` - invokes the given callback on the partition that owns the given key
```java
invokeBy(key, service -> service.put(key, value));
```

{% include common-links.html %}
