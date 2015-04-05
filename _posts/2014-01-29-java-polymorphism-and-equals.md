---
layout: article
title: "Java Polymorphism And equals()"
categories: articles
date: 2014-01-29
modified: 2014-01-29
tags: [java,core,polymorphism]
image:
  feature: 
  teaser: /2014/01/equals.jpg
  thumb: 
ads: false
excerpt: How should equals() behave in case of polymorphic classes? What are the pitfalls here?
comments: true
---

Let's start by having a small introduction. The mother of all Java classes -- `Object` -- does define a `equals()` method, which is meant to return true if the passed instance is equal to the current instance. The default implementation is limited to comparing references. So it will only return `true` when the current and the passed object are the same. The `equals()` method is meant to be overridden by extending classes with meaningful logic. The implementation has to obey some requirements (from javadoc on [Object.equals()]):

> The equals() method guarantees that...
>
> * It is reflexive: for any non-null reference value `x`, `x.equals(x)` should return `true`.
* It is symmetric: for any non-null reference values `x` and `y`, `x.equals(y)` should return `true` if and only if `y.equals(x)` returns `true`.
* It is transitive: for any non-null reference values `x`, `y`, and `z`, if `x.equals(y)` returns `true` and `y.equals(z)` returns `true`, then `x.equals(z)` should return `true`.
* It is consistent: for any non-null reference values `x` and `y`, multiple invocations of `x.equals(y)` consistently return `true` or consistently return `false`, provided no information used in equals comparisons on the objects is modified.
* For any non-null reference value `x`, `x.equals(null)` should return `false`.

Quite a bunch. Luckily those requirements are often easy to achieve. Let's create a simple class and equip it with a `equals()` method.

## A Drink
{% highlight java %}
public class Drink {
  private final int size;

  public Drink(final int size) {
    this.size = size;
  }

  @Override
  public boolean equals(final Object obj) {
    if (!(obj instanceof Drink)) return false;
    return equals((Drink) obj);
  }

  public boolean equals(final Drink other) {
    return this.size == other.size;
  }
}
{% endhighlight %}

This `equals()` implementation obeys all the requirements above. And I added a convenience method with the exact type. Note that `equals(Drink)` does overload `Object.equals(Object)` but it [does not override] it! The difference between those two will be important later.

## A Drink? A Coffee? A Coke?
Now let's introduce [polymorphism]. We add two classes Coffee and Coke which extend the `Drink` class:

{% highlight java %}
public class Coffee extends Drink {
  private final int coffeine;

  public Coffee(final int size, final int coffeine) {
    super(size);
    this.coffeine = coffeine;
  }

  @Override
  public boolean equals(final Object obj) {
    if (!(obj instanceof Coffee)) return false;
    return equals((Coffee) obj);
  }

  public boolean equals(final Coffee other) {
    if (!super.equals(other)) return false;
    return coffeine == other.coffeine;
  }
}

public class Coke extends Drink {
  private final int sugar;

  public Coke(final int size, final int sugar) {
    super(size);
    this.sugar = sugar;
  }

  @Override
  public boolean equals(final Object obj) {
    if (!(obj instanceof Coke)) return false;
    return equals((Coke) obj);
  }

  public boolean equals(final Coke other) {
    if (!super.equals(other)) return false;
    return sugar == other.sugar;
  }
}
{% endhighlight %}

The `equals()` methods are implemented here in a similar way. Everything looks fine, doesn't it? Let's see how the `equals()` methods behave:

{% highlight java %}
final Drink drink = new Drink(15);
final Drink secondDrink = new Drink(15);

System.out.println("drink.equals(secondDrink): " + drink.equals(secondDrink));
System.out.println("secondDrink.equals(drink): " + secondDrink.equals(drink));

final Coffee coffee = new Coffee(15, 3);
final Coke coke = new Coke(15, 42);

System.out.println("coffee.equals(drink): " + coffee.equals(drink);
System.out.println("drink.equals(coffee): " + drink.equals(coffee));
System.out.println("coke.equals(coffee): " + coke.equals(coffee));
{% endhighlight %}

What’s the output? Your brain might tell you something like this:

~~~
drink.equals(secondDrink): true
secondDrink.equals(drink): true
coffee.equals(drink): false
drink.equals(coffee): false
coke.equals(coffee): false
But what’s the actual output? This:
drink.equals(secondDrink): true
secondDrink.equals(drink): true
coffee.equals(drink): true
drink.equals(coffee): true
coke.equals(coffee): true
~~~

Wow! What's happening? `drink.equals(coffee)` is passed a parameter of type `Coffee`. The best method match for this type is `Drink.equals(Drink)`. This method does only compare the size field. Since it's equal it returns true. `coffee.equals(drink)` is passed a parameter of type `Drink`. The best method match for this type is.... `Drink.equals(Drink)`! Not `Coffee.equals(Object)`! So again only the size field is compared. The same goes for `coke.equals(coffee)`: `Drink.equals(Drink)` is invoked.

**First lesson**: it’s a bad idea to implement convenience public equals() methods with different types than Object. Or in other words: do not overload equals(), override it.

Now let's "fix" this problem by making the overloaded methods private. What will be the output this time? This:

~~~
drink.equals(secondDrink): true
secondDrink.equals(drink): true
coffee.equals(drink): false
drink.equals(coffee): true
coffee.equals(coke): false
~~~

Still not quite the output we're expecting. What's happening now? When `coffee.equals(drink)` is invoked, the `Coffee.equals(Object)` method is executed, `Drink` instance is checked against `instanceof Coffee` and this evaluates to `false`. But when we invoke `drink.equals(coffee)` the `equals()` implementation in `Drink` is executed and the passed instance is checked against `instanceof Drink`. Since `Coffee` is a extension of `Drink`, this evaluates to `true`.

## Not all Drinks are Coffees
So is polymorphism broken in Java? Not quite. It seems like `instanceof` is not the check you should use per default in `equals()`. It's sometimes important, we'll see in a minute, but usually what you'd like to do is to use `Object.getClass()` and compare the class of the passed instance to the class of the current instance:

{% highlight java %}
public class Drink {
  ...

  @Override
  public boolean equals(final Object obj) {
    if (obj == null) return false;
    if (this.getClass() != obj.getClass()) return false;
    return equals((Drink) obj);
  }

  private boolean equals(final Drink other) {
    return this.size == other.size;
  }
}

// changes in Coffee and Coke similar
{% endhighlight %}

As specified by [documentation of getClass()], it is guaranteed to return the same `Class` instance for the same class. So it is save to use the `==` operator. Note that `obj.getClass()` is compared against `this.getClass()` and not against `Drink.class`! If we'd compare against the hard coded class `super.equals()` invocations from extending classes would always fail for non `Drink` instances.

**Second lesson**: use `Object.getClass()` in `equals()` if unsure. Only use `instanceof` if you know what you do. ;)

## Instanceof
Now when would one want to use `instanceof` in `equals()` implementations? The semantics of `equals()` implementations are actually up to you (as long you follow the restrictions stated at the beginning). In my example above I wanted to have `Drink!=Coffee!=Coke`. But that's just a definition thing. Sometimes you want to have a set of types behave like one type. The Java class library does this for lists and maps for example. A `TreeMap` and a `HashMap` are considered equal if they contain the same objects. Even though a `TreeMap` has an element order which a `HashMap` does not have. The types achieve this by having a `AbstractMap` class implement a [equals()] method which checks against `instanceof Map` and checks only `Map` properties. All extensions of `AbstractMap` do not override (and do not overload) `equals()`.

## One more thing
Don't forget to implement `hashCode()` if you override `equals()`. Both methods have a tight relationship. Whenever `a.equals(b)` returns true, also `a.hashCode()` has to be equal to `b.hashCode()`!

[Object.equals()]: http://docs.oracle.com/javase/7/docs/api/java/lang/Object.html#equals%28java.lang.Object%29
[does not override]: http://stackoverflow.com/questions/10568772/overloaded-and-overridden-in-java
[polymorphism]: http://en.wikipedia.org/wiki/Polymorphism_%28computer_science%29
[documentation of getClass()]: http://docs.oracle.com/javase/7/docs/api/java/lang/Object.html#getClass%28%29
[equals()]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/util/AbstractMap.java#AbstractMap.equals%28java.lang.Object%29
