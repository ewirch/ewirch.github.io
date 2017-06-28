---
layout: article
title: "Be Careful When Combining Collections of Different Type"
categories: articles
date: 2015-04-12
modified: 2015-04-12
tags: [java, core, collections, pitfall]
image:
  feature: 
  teaser: /2015/04/collection.jpg
  thumb: 
ads: false
comments: true
---

Sometimes the runtime speed of collection methods can vary extremely when called with different `Collection` implementations. But why?


Have a look at this code:

{% highlight java %}
Set<Integer>        set1 = getHashSetHaving5000Elements();
Collection<Integer> c1   = getArrayListHaving5001Elements();
 
Set<Integer>        set2 = getHashSetHaving5000Elements();
Collection<Integer> c2   = getArrayListHaving4999Elements();

set1.removeAll(c1);
set2.removeAll(c2);
{% endhighlight %}

Which block will be faster? 'The second one!', you yell, 'It contains two elements less!'. That's true. But it will surprise you to know that it is more than twice as fast, not just a couple of milliseconds! How come? The solution is in the documentation of `AbstractSet.removeAll()`:

> This implementation determines which is the smaller of this set and the specified collection, by invoking the size method on each. If this set has fewer elements, then the implementation iterates over this set, checking each element returned by the iterator in turn to see if it is contained in the specified collection. If it is so contained, it is removed from this set with the iterator's remove method. If the specified collection has fewer elements, then the implementation iterates over the specified collection, removing from this set each element returned by the iterator, using this set's remove method.

So if the passed collection is bigger, even by one element, the implementation will call `c1.contains()` for each element of the set -- which is pretty slow for `ArrayList`. That's surprising, isn't it? Here's the code of `AbstractSet.removeAll()`:

{% highlight java %}
public boolean removeAll(Collection<?> c) {
	Objects.requireNonNull(c);
	boolean modified = false;
	
	if (size() > c.size()) {
	    for (Iterator<?> i = c.iterator(); i.hasNext(); )
	        modified |= remove(i.next());
	} else {
	    for (Iterator<?> i = iterator(); i.hasNext(); ) {
	        if (c.contains(i.next())) {
	            i.remove();
	            modified = true;
	        }
	    }
	}
	return modified;
}
{% endhighlight %}

I understand why the designers of `AbstractSet` did this 'optimization'. After all you can't know how both instances (the `AbstractSet` instance and the `Collection` instance) are implemented. But I'd have overridden this method in `HashSet`, and would have added some black list checks for types which are known to be slower than `HashSet`.
