---
layout: article
title: "Resolving circular dependencies in C++"
categories: articles
date: 2013-12-15
modified: 2013-12-15
tags: [c++, code style]
image:
  feature: 
  teaser: /2013/12/circular.jpg
  thumb: 
ads: false
comments: true
---
I stumbled several times already upon situations where I created a circular dependency between two classes. A circular dependency is bad design in general.

It increases coupling between classes and thus makes changes more difficult ([Circular dependency - Wikipedia][wikipedia]). But in some languages (for example Java) it's possible to create classes which depend on each other without compiler errors. So very often circular dependencies are created and used without noticing. Differently for C++. The strict processing order of C++ compilers will spill out a bunch of errors when you try to make two classes depend on each other.
But you surely had a reason for this attempt. So how you achieve your aim whilst avoiding a circular dependency? See this example of a thread safe, usage based, load balanced object pool implementation:

{% highlight c++ %}
// File: ObjectPool.h

#pragma once

#include <list>
#include <stdlib.h>
using namespace std;

// provides basic locking capabilities
#include "Lock.h"

// object type supplied by the pool
#include "Element.h"

class ObjectPool {
private:
 list<Element*> elements_;
 Lock lock_;

public:
 ObjectPool(size_t size) {
  for (size_t i = 0; i < size; i++) {
   elements_.push_back(new Element());
  }
 }

 Element *take() {
  // acquires the pool lock. AutoLockAndRelease will release the lock when scope is left
  AutoLockAndRelease autoLock(lock_);

  Element *chosen = getLeastUsedElement();
  chosen->incUsageCount();
  return chosen;
 }

 void release(Element *element) {
  AutoLockAndRelease autoLock(lock_);

  assertThisIsOurElement(element);
  element->decUsageCount();
 }

private:
 // checks if this element belongs to this pool
 void assertThisIsOurElement(Element *element) {
  // implementation details omitted
 }

 Element *getLeastUsedElement() {
  // implementation details omitted
 }
};
{% endhighlight %}


{% highlight c++ %}
// File: Element.h

#pragma once

class Element {
private:
 int usageCount_;

public:
 Element():usageCount_(0) {
 }

 void incUsageCount() {
  usageCount_++;
 }

 void decUsageCount() {
  usageCount_++;
 }

 int getUsageCount() {
  return usageCount_;
 }
};
{% endhighlight %}


This is how it is used:

{% highlight c++ %}
ObjectPool pool(5);
Element *element = pool.take();
// use element
pool.release(element);
{% endhighlight %}

I spare you the implementation details of `Lock.h`. Just assume it contains a lock implementation capable of synchronizing threads.

I'd like to modify this object pool and enable the element to release itself. I'd like to use the pool like this:

{% highlight c++ %}
ObjectPool pool(5);
Element *element = pool.take();
// use element
element.release();
{% endhighlight %}

Here are the code changes implementing this feature:

{% highlight c++ %}
// File: ObjectPool.h

 ...
public:
 ObjectPool(size_t size) {
  for (size_t i = 0; i < size; i++) {
   elements_.push_back(new Element(*this));
  }
 }

 ...
{% endhighlight %}


{% highlight c++ %}
// File: Element.h
...
class Element {
private:
 int usageCount_;
 ObjectPool &parent_;

public:
 Element(ObjectPool &parent):usageCount_(0), parent_(parent) {
 }

 ...

 void release() {
  parent_.release(this);
 }
};
{% endhighlight %}

This example suffers from two problems: 

1. The obvious one: The example doesn't compile. My compiler issues the error `'ObjectPool' does not name a type`. This arises from the attempt to create a circular dependency: `ObjectPool` depends on `Element`, and `Element` depends on `ObjectPool`.
1. A a very subtle problem: The reference to the `ObjectPool` instance is made public to the `Entry` instance before the constructor of the `ObjectPool` class finished initializing the instance. This is known as early reference leak and should be avoided ([Java theory and practice: Safe construction techniques][safe-constructor-techniques], [Should you use the this pointer in the constructor?][should-i-use-this-pointer]).

// FIXME: "first"? the second problem is not touched

Let's first have a look at what we actually are trying to achieve here. The ObjectPool class manages a specified amount of entries. For each entry a usage count is maintained. The user can call `take()` to receive the entry with the lowest usage count. The `ObjectPool` class will use a lock to synchronize `take()` and `release()` invocations so it can be used safely in a threaded context. Once the user is done using a entry he invokes `release()` on the entry which then invokes `release()` on its parent which does a synchronized release. So how do we resolve the circular dependency while preserving the logic?

The solution is to decouple the `ObjectPool` and the `Element` class. I'll show here how to decouple both using a interface. The idea is, the `Element` class doesn't has to know the whole implementation of the `ObjectPool` class. It only needs to know the `release()` method. So we create a interface containing the release method and let `ObjectPool` implement the interface. Then we tell `Entry` only about the interface but not the `ObjectPool` class.

Here is the interface:

{% highlight c++ %}
// File: Releaser.h

#pragma once

template<typename T>
class Releaser {
public:
 virtual ~Releaser() {};
 virtual void release(T *element) = 0;
};
{% endhighlight %}

It defines a single release method with a template type. The template type is essential here. Without the use of a template the interface would need to include `Element` type to use it as parameter type. But this would again create a circular dependency: `ObjectPool -> Releaser -> Element -> Releaser`. Using the template trick we break the cycle. Here are the modified implementations of `ObjectPool` and `Element`.

{% highlight c++ %}
// File: ObjectPool.h

#pragma once

#include 
#include 
using namespace std;

#include "Lock.h"
#include "Element.h"
#include "Releaser.h"

// FIXME: template parameter missing, virtual destructor missing
class ObjectPool: public Releaser {
private:
 list elements_;
 Lock lock_;

public:
 ObjectPool(size_t size) {
  for (size_t i = 0; i < size; i++) {
   elements_.push_back(new Element(*this));
  }
 }

 Element *take() {
  AutoLockAndRelease autoLock(lock_);

  Element *chosen = getLeastUsedElement();
  chosen->incUsageCount();
  return chosen;
 }

 void release(Element *element) {
  AutoLockAndRelease autoLock(lock_);

  assertThisIsOurElement(element);
  element->decUsageCount();
 }

private:
 void assertThisIsOurElement(Element *element) {
  // implementation details omitted
 }

 Element *getLeastUsedElement() {
  // implementation details omitted
 }
};
{% endhighlight %}


{% highlight c++ %}
// File: Element.h

#pragma once

#include "Releaser.h"

class Element {
private:
 int usageCount_;
 Releaser &releaser_;

public:
 Element(Releaser &releaser):usageCount_(0), releaser_(releaser) {
 }

 void incUsageCount() {
  usageCount_++;
 }

 void decUsageCount() {
  usageCount_++;
 }

 int getUsageCount() {
  return usageCount_;
 }

 void release() {
  releaser_.release(this);
 }
};
{% endhighlight %}

Now `Element` only references `Releaser` and the dependency cycle is broken.

[wikipedia]: http://en.wikipedia.org/wiki/Circular_dependency
[safe-constructor-techniques]: http://www.ibm.com/developerworks/java/library/j-jtp0618/index.html
[should-i-use-this-pointer]: http://www.parashift.com/c++-faq/using-this-in-ctors.html
