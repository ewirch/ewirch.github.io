---
layout: article
title: "Do We Have To Synchronize Everything?"
categories: articles
date: 2014-03-24
modified: 2014-03-24
tags: [java, concurrency, core, jmm, memory-model, synchronization, threads]
image:
  feature: 
  teaser: /2014/03/synchronize.jpg
  path: /2014/03/synchronize.jpg
  thumb: 
ads: false
comments: true
---

Evaluation of [ThreadSafe][] made me think about the Java Memory Model and its implications on threaded programs.


{% include toc.html %}

## The Problem

In tech gibberish the Java Language Specification states that a thread is only guaranteed to see memory values modified by other threads if it synchronizes to them. It is not enough if only the writing thread uses a synchronize statement. Each thread which needs to see up to date values, needs to synchronize (enter the synchronized statement). Even more restrictive: thread T1 is only guaranteed to see all changes T2 did, before releasing monitor L, when it also acquires monitor L before reading the values. In JLS language ([JLS 17.4.4][]):

> An unlock action on monitor `m` synchronizes-with all subsequent lock actions on `m` (where 'subsequent' is defined according to the synchronization order).

Example (`globalInt == 0` at the beginning):

```text
T1                                  T2
_______________________________________________________________________
globalInt = 3;
                                    int x = globalInt; // may be 0 or 3
```

The Java Memory Model does not define what T2 is going to see. Now with synchronization but different monitors:

```text
T1                                  T2
_______________________________________________________________________
synchronized (L1) {
  globalInt = 3;
}
                                    int x;
                                    synchronized (L2) {
                                      x = globalInt; // may be 0 or 3
                                    }
```

Even though it uses synchronization and even if T2 runs after T1 finished, T2 is not guaranteed to see the new value. The JMM guarantees this only for the case when synchronizing on the same monitor (volatile and final are also possible, they will be discussed later):

```text
T1                                  T2
_______________________________________________________________________
synchronized (L1) {
  globalInt = 3;
}
                                    int x;
                                    synchronized (L1) {
                                      x = globalInt; // will see 3, if
                                                     // executed after
                                                     // T1
                                    }
```

If that's true, what does this mean to a real case scenario like "initialize once, use multiple times":

```java
private static volatile Map<String> map;
 
public Map<String> getMap() {
    if (map == null) {
        synchronized (this) {
            if (map == null) {
                Map<String> newMap = new HashMap<>();
                fillMap(newMap);
                map = newMap;
            }
        }
    }
    return map;
}
```

This is a map lazily initialized by a classic [double-checked][] idiom -- correctly implemented as of Java 1.5[^1]. This example uses a `HashMap` instead of a `ConcurrentMap` for optimization purposes. Since the map is only read after initialization, this is save. `volatile` makes sure other threads are going to see the new reference written to the variable map. But what happens to the elements of the map? The map reference is volatile, but the map elements are not. You learned previously that two threads need to synchronize to the same monitor if they want to make sure to see the same values. But in this case some readers may never reach the synchronize statement if the initialization was finished already. So what are the readers guaranteed to see?


[^1]: See [Is The Double-Check Idiom Really *Really* Fixed?][double-checked-fixed] for a description on why `map` has to be `volatile`.

## Java Memory Model -- Enlightened

### Volatile

The promise the Java Memory Model makes for locks also goes for `volatile` reads and writes. It defines, that all writes to a volatile variable `synchronize-with` all reads of the same variable ([JLS 17.4.4][]). This means, once the reader threads read the variable `map`, they are guaranteed to see all changes the writer thread did before writing to `map`. This means, assigning the new `HashMap` instance to a local variable (`newMap`) first and then assigning it to the field (only after it is fully initialized), is crucial for two reasons:

1. Assigning `map` before `fillMap()` would reveal the reference to the newly created map to other threads before initialization was finished. This means other threads could see inconsistent data. Additionally this might lead to serious problems when `get()` and `put()` are executed concurrently (`HashMap` is not thread safe).
1. The JMM guarantees visibility only for writes which happened before the write to a `volatile` field. This means all writes after assignment of `map` are not guaranteed to be visible to other threads.


### Non-Volatile

There is another way to make it work according to the JMM if the object doesn't has to be created lazily. The JLS says ([JLS 17.5][]): 

> An object is considered to be `completely initialized` when its constructor finishes. A thread that can only see a reference to an object after that object has been completely initialized is guaranteed to see the correctly initialized values for that object's final fields.

So may you rewrite the example above in this way?:

```java
private static final Map<String> map = new HashMap<>();
 
public Map<String> getMap() {
    if (map.size() == 0) {
        synchronized (this) {
            if (map.size() == 0) {
                fillMap(map);
            }
        }
    }
    return map;
}
```

No, you may not rewrite it like this. Apart from the obvious problem that, the map could be modified while it is being read (once the first element was added), there are no data visibility guarantees according to the JMM regarding the map elements. Since the `map` field is not `volatile` any more, there is no `synchronizes-with` relationship between threads any more. But this will work:

```java
private static final Map<String> map = new HashMap<>();
private static volatile initialized = false;

public Map<String> getMap() {
    if (!initialized) {
        synchronized (this) {
            if (!initialized) {
                fillMap(map);
                initialized = true;
            }
        }
    }
    return map;
}
```


Now reading of `initialized` `synchronizes-with` the write of `initialized` variable and thus all writes happened until that moment. But then we're back at using `volatile` fields. The following approach works the best, when you can go completely without lazy initialization:

```java
private static final Map<String> map = createAndFillMap();

public Map<String> getMap() {
    return map;
}
```

The next sample works as well *AND* is lazy initialized, but it tries to be smart and thus should not be your first choice ([Don't Be Too Smart][]).

```java
public Map<String> getMap() {
    return MapHolder.map;
}
 
private static class MapHolder {
    private static final Map<String> map = createAndFillMap();
}
```

This implementation relies on the JLS guarantee, that classes are loaded when they are used for the first time ([JLS 5.3][]).


But we're still not done talking about finals. If you read the JLS guarantee for final fields carefully you noticed this part: 

> ... after that object has been `completely initialized` is guaranteed to see the correctly initialized values for that object's final fields.

`Completely initialized` is defined by finishing the constructor. Thus if the constructor leaks the reference to the object being constructed (`this`), there are no guarantees about thread visibility of the `final` fields. Example leaking `this`:

```java
public class Counter {
    private final AtomicInteger counter;

    public Counter(int startValue) {
        counter = new AtomicInteger(startValue);
        CounterRegistry.register(this);
    }
}
```

The constructor above leaks `this` reference before the constructor is finished. If a foreign thread picks up the reference (before the constructor finished) it may or may not see correct values. In general: avoid leaking `this` from constructors.


## ReadWriteLock Implementation

Just out of curiosity: how is `ReadWriteLock` implemented? After all this lock features distinct handling of write-locking and read-locking. If these are separate locks, how does this comply to the JMM, if the same monitor has to be used?

The `ReentrantReadWriteLock` implementation of `ReadWriteLock` uses two local fields `readerLock` and `writerLock` which internally use the same `Sync` object. `Sync` is a implementation of `AbstractQueuedSynchronizer`. And `AbstractQueuedSynchronizer` in turn uses a internal `volatile` field. So it boils down to: `ReentrantReadWriteLock` is implemented -- in perfect harmony with the JMM -- using one `volatile int` field (using `LockSupport.park()` to wait if acquiring write lock doesn't succeed immediately).

## Compiler Optimizations Allowed By the Java Memory Model
The JMM preserves great freedom for optimizations of compilers. The whole JMM guarantees build around `happens-before`, `synchronizes-with` relationships and `well-formed execution` rules. The definitions go like "a read has to see the effects of a write, if that write came before the read in program order, and there was no other write in between". It doesn't say that the write actually has to happen when the write command is encountered in program order. It only states that the read has to see the effects. So it's completely valid for the compiler to move the write just immediately before the reading line. Or: let the write happen only to processor registers and write it back to memory much later when the compiler thinks it's appropriate. Or even: remove the write completely if there is no read which needs to see the write effects.

```java
x = 0;
y = 1;
```

It wouldn't surprise anyone if the compiler would reorder the two statements. There is probably no optimization benefit, but there is also no obvious reason why the compiler shouldn't. But take this code:

```java
// double-checked idiom implemented wrongly
private Object instance;
Object getInstance() {
    if (instance == null) {
        synchronized(this) {
            if (instance == null) {
                Object helper;
                synchronized (this) {
                    helper = new Object();
                }
                instance = helper;
            }
        }
    }
    return instance;
}
```

(the code is discussed in [Bidirectional Memory Barrier][] as a attempt to implement the double-checked idiom without volatile keyword) The Java Memory Model does not prevent the compiler to change the code to:

```java
private Object instance;
Object getInstance() {
    if (instance == null) {
        synchronized(this) {
            if (instance == null) {
                synchronized (this) {
                    Object helper;
                    helper = new Object();
                    instance = helper;
                }
            }
        }
        }
    return instance;
}
```


And then, in the next optimization step:

```java
private Object instance;
Object getInstance() {
    if (instance == null) {
        synchronized(this) {
            if (instance == null) {
                synchronized (this) {
                    instance = new Object();
                }
            }
        }
    }
    return instance;
}
```

There are rules which prevent the compiler to move lines, which are inside a synchronized block, out of the block. But there is no rule which forbids to move lines inside the synchronized block. Surprising, isn't it?

The lesson from this is: don't try to be too smart. (again?! ;) ) Stick to this basic rules of the Java Memory Model which are: If there is something which can be accessed by multiple threads, then:

* make it `final`, OR
* make it `volatile`, OR
* use the same monitor to coordinate access to the data.


## Hard Side of Live (Hardware)

Until now I talked only about theoretical guarantees the JMM offers. This gives the freedom to the Java developer to code against one memory model. Remember: Java is designed to run everywhere. If Java wouldn't offer something like a JMM, the developers would need to bother themselves with all the difficulties and pitfalls of different architectures. But how is the JMM applied to a specific architecture, let's say: x86?

To recapitulate: we were concerned with 'visibility' of updated memory values. We talked about threads not seeing new values, because they still use the (old) cached values. The cure in terms of JMM was to use `volatile` or synchronized.

A lot of people think when `volatile` is written to, or when a synchronized block is left, CPU caches are flushed, so all values will be reloaded. But in fact there is no CPU operation like "flush the cache". All modern x86 CPUs try very hard to keep the CPU caches transparent and make the memory appear consistent to the developer. They do this by implementing cache coherency. So memory writes are automatically detected and updated in all caches. And: this also only applies to multi processor systems, or processors having a memory cache for each core. For a single CPU, single core system each thread ultimately sees the newest values (once they have been written to memory).

So does this mean for x86 architecture the JMM is not necessary? Does it add unnecessary synchronization statements or is it NOPed out[^2] when compiled? Far from it! Even with cache coherency the JMM is required. Required to:

* guarantee read/write order
* atomicy when writing/reading values which cannot be written/read atomically
* get "caches" in line you probably even didn't think of: registers
* offer guarantees even in case of optimizations applied on top of the memory cache.

[^2]: Replaced by "no operation" instructions.


### Memory Access Reorderings

Memory access can be reordered. It can be reordered by the compiler or the CPU. Reordering by the compiler was already noted earlier. There are some restrictions to reordering introduced by the JMM, but compilers still have a lot of freedom to change the execution order compared to program order (as in the source). So when we have code like

```java
x = 3;
written = true;
```

nothing prevents the compiler to reorder these statements like this, making this code fail:

```java
while (!written) wait();
assert x == 3;
```

But even when the compiler did not change the order, the CPU might change it. Modern CPUs try to parallelize as much as possible. When code is executed in parallel it may appear to run out of order. Take for example a floating point calculation, a store of the calculation result to memory, and a subsequent load of another variable from memory:

```java
float f2 = f1 * 4.38473723f;
if (x == 3) { ... }
```

The load of `x` might be executed in parallel to the floating point calculation. So the value of `x` might be read from memory while `f2` was still not written to memory yet.

### Atomic Writing/Reading of Values
Some Java data types can be written and read by the processor in one operation. For example a `int` on a 32 bit system. While other data types require multiple operations. For example a `long` (64 bit) on a 32 bit system. Having two operations to write a value allows other processors to observe a half written (thus inconsistent) value. For variables declared `volatile` Java needs to make sure the variable appears to be written and read atomically.

### Registers
There are more types of "cache" than the usual CPU memory cache everyone thinks of when someone says "cache". The Java compiler could optimize code by storing variables temporarily in CPU registers. This can be considered a cache too. Take this code:

```java
for (int i = 0; i < 1000; i++) {
    j = j + 10;
}
```

It is almost certain that the variable `i` will only exist in CPU registers. It's also very likely that `j` will be loaded to a CPU register at the beginning of the loop, and only written back to memory when the loop finishes. No other threads will be able to observe the intermediate steps applied to `j`. If one needs to make sure other threads will observe the changes, he needs to tell this to Java explicitly.


### Optimizations of the Cache

In their effort to speed up CPU memory access, CPU designers applied even optimizations to the cache, which is actually a optimization to optimization. There are so called store buffers. They are used to queue stores to memory applied by the processor. Using those, the processor can continue its work without to have to wait for the store operation to complete. With store buffers in use, a couple of things can happen: 

* The store operation itself is delayed. This means some writes/reads may appear out of order.
* There are no guarantees in which order values in the store buffer are going to be written to memory. It's possible that, if two variables lie next to each other in memory, they are written in one operation, even if there were other store operations in between.

And there are invalidation queues. A little background on this: One way how CPUs implement cache coherency is to use the MESI cache coherency protocol. MESI stands for Modified Exclusive Shared Invalid and names the states cache entries may have. When a CPU needs to modify a variable, it sends a invalidation message to other CPUs. The others mark the entries in their caches as invalid (if they store the entry in their cache at all). The modifying CPU needs to wait until all CPUs confirmed the invalidation message. This takes time. A lot of time in CPU processing terms. So invalidation queues were introduced. Each CPU immediately acknowledges a invalidation message and stores a entry in its invalidation queue. The queue is processed later on. This means there is some time between a invalidation message and the message being applied in all caches. So it's possible for CPU0 to process its store buffers and update all values in memory, while CPU1 still has not yet processed the invalidation queue. So CPU1 could read a old value for a variable from its cache while the invalidation queue is still not yet processed.

So what does Java do to guarantee consistency in all those cases? Java utilizes so called memory barriers. In simple terms a memory barrier forces those queues (store buffers, read buffers, invalidation queue) to run dry before execution can continue. When a `volatile` variable is written, a single cache entry is invalidated and the invalidation queue and store buffers are processed.


### Lessons We Learned
Consistency and performance are conflicting. ;) But you knew this already, right? There are a lot of subtle things going on. And the magic spell for the Java developer to handle all this is the Java Memory Model. The JMM is a nice thing to rely on, facing the amount of architectures the code could be executed on.



*[JMM]: Java Memory Model
*[JLS]: Java Language Specification

[ThreadSafe]: http://www.contemplateltd.com/threadsafe
[JLS 17.4.4]: http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.4
[JLS 17.5]: http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.5
[JLS 5.3]: http://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.3
[double-checked]: http://en.wikipedia.org/wiki/Double-checked_locking
[double-checked-fixed]: {% post_url 2014-02-27-is-double-check-really-fixed %}
[Don't Be Too Smart]: http://programmer.97things.oreilly.com/wiki/index.php/Don%27t_Be_Too_Smart
[Bidirectional Memory Barrier]: http://www.cs.umd.edu/~pugh/java/memoryModel/BidirectionalMemoryBarrier.html
