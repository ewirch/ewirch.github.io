---
layout: article
title: "Is The Double-Check Idiom Really *Really* Fixed?"
categories: articles
date: 2014-02-27
modified: 2014-02-27
tags: [java,core,cuncurrency,memory-model]
image:
  feature: 
  teaser: /2014/02/doublecheck.jpg
  thumb: 
ads: false
comments: true
---

The double-check idiom is a way to reduce lock contention for a lazy initialized thread safe class. Unfortunately it used not to work. Luckily it was fixed. But under which conditions is it to be considered fixed?


Preface: There is a magnificent article covering this topic: [Double Check Locking by Bill Pugh et al.][bill-pugh]. This article tries to rephrase the facts from the linked resource in a simpler way. Anyone interessted in the deep core details should read the article by Bill Pugh et al.

## Java 1.4
When Java 1.4 was the most recent release the so called Double-Check Idiom was considered broken. Due to the Java memory model specifications it was not guaranteed to work as one would expect. This is the double-check idiom in it's pure form:

{% highlight java %}
piblic class Boss {
  private String name;
  
  public Boss() {
    this.name = "";
  }
}

public class Company {
  private Boss boss = null;
  
  public Boss getBoss() {
    if (boss == null) {
      synchronized (this) {
        if (boss == null) {
          boss = new Boss();
        }
      }
    }
    return boss;
  }
}
{% endhighlight %}

There are two major reasons why this could fail:

1. operation reordering,
2. memory caches.

**Operation Order:** The Java memory model guarantees that all memory operations will be finished before the synchronized block is left. But it doesn't say anything about the order of the memory operations inside the synchonized block. A compiler might change the order of memory operations. If the constructor of `Boss` is inlined, the assignment of boss field (pointing to memory holding boss instance) could be executed before the instance fields of the `Boss` class are assigned by the constructor code. This means a concurrent thread could see `boss!=null` while the initialization is still not finished.

**Memory Caches:** Each thread may have it's own local cache of the main memory. So even if the initializing thread did finish all memory write operations, a concurrent thread might see the new value of the `Company.boss` field but the old (uninitialized) memory values for the `Boss` class fields. This is what the Java Language Specification (Java 1.7) says about memory effects of the synchronized block:

> JLS 17.4.4. Synchronization Order
>
> ... An unlock action on monitor `m` synchronizes-with all subsequent lock actions on `m` (where "subsequent" is defined according to the synchronization order). ...

So it guarantees that everything what thread A did before it left the synchronized block will be visible to thread B when it enters a synchronized block which locks on the same mutex. Note that it doesn't state anything about what is visible to threads which do *not* enter a synchronized block! So the changes from the double-checked idiom might be visible, might be partially visible or might be not visible to other threads at all.

## Changes to the Java Memory Model in 1.5
Java 1.5 implements a more recent memory model specification. The modification which is interesting in this context is the change to access to volatile variables. The read or write of a volatile variable is not allowed to be reordered with respect to any previous or following read or writes. This means the compiler is not allowed to reorder the write of `Company.boss` field if it is declared volatile.

The fixed example from above would look like this:

{% highlight java %}
public class Company {
  private volatile Boss boss = null;
  
  ....
}
{% endhighlight %}

Concluding: the double-checked idiom was really really broken before Java 1.5. It is really really fixed with Java >= 1.5 only when the the field being checked in the double-checked idom is declared volatile. If it is not, it's still broken.


[bill-pugh]: http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html