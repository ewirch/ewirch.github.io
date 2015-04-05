---
layout: article
title: "It’s not my code! I googled it."
categories: articles
date: 2013-11-13
modified: 2013-11-13
tags: [code style, eclipse, java, sonar]
image:
  feature: 
  teaser: /2013/11/its-not-my-code.jpg
  thumb: 
ads: false
comments: true
---

Actually this blog post is not about Google. And it is even not about copying code. But it is indeed about code which I did not write.


Since a couple days we use [Sonar](http://www.sonarsource.org/) to check our code for coding style or conventions violations. Well this is not that new. We used to use Checkstyle, PMD, Findbugs already for a couple of years. But the switch to Sonar brought the heap of violations in our code up my mind again. Sonar says we have around 3000-5000 violations in our projects. Probably the most of them are eligible. But some of them are not:

![equals fail]({{site.url}}/images/2013/11/equals-fault.png)

Here you see some of the violations found in the `Configuration` class in the `equals()` method. Nearly each line has a violation. The problem is: `equals()` is a automatically generated method. Coding conventions violations in generated code are just useless. Generated code doesn’t has to be maintainable. It doesn’t has to be readable.

I thought about how to tell Sonar to ignore this code. One could use the `//NOSONAR` comment to make Sonar ignore lines. But you’d have to place it on every line. Or you could use

{% highlight java %}
	@SuppressWarnings("all")
{% endhighlight %}

but this would suppress all warnings, not only Sonar violations ([reference](http://jira.codehaus.org/browse/SONAR-1760)).

Then I stumbled upon @Generated annotation which is [part of Java since 1.6](http://docs.oracle.com/javase/6/docs/api/javax/annotation/Generated.html). Using this annotation, code generators could automatically mark generated code, making life easier for code analyzers and developers. So in a perfect world my Eclipse code generator would generate this method:

{% highlight java %}
@Override
@Generated("Eclipse source generator")
public boolean equals(final Object obj) {
    if (this == obj) return true;
    if (obj == null) return false;
    if (getClass() != obj.getClass()) return false;
    final ConfigurationBase other = (ConfigurationBase) obj;
    if (autoRefreshDatabaseEnabled != other.autoRefreshDatabaseEnabled) return false;
    ....
}
{% endhighlight %}

and all code analyzers would magically ignore this method.

This idea was already [picked up by the sonar team](http://jira.codehaus.org/browse/SONARJAVA-71). But sadly it's not yet implemented.
