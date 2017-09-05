---
layout: article
title: "Memory leaks even with WeakReferences"
categories: articles
date: 2013-12-18
modified: 2013-12-18
tags: [java, memory, memory leaks, pitfalls]
image:
  feature: 
  teaser: /2013/12/leak.jpg
  path: /2013/12/leak.jpg
  thumb: 
ads: false
comments: true
---

A crash report arrived at my desk the other day. The system crashed because it ran out of memory. And the major memory consumer was a `WeakHashMap`. Very interesting, since `WeakHashMaps` are usually free memory when the content is not needed any more.


First some background. Imagine you've build a JSP page which generates a HTML page with a lot of URLs on it. You construct those URLs from different parameters. You use `URLEncoder` since you need valid URLs independent of the URL parameter contents you print. Once everything works fine, you realize that your URLs share many strings. So `URLEncoder` is called unnecessarily very often. You try to optimize the situation by creating a cache for `URLEncoder`:

```java
class CachedUrlEncoder {
 static private Map<String, String> encodedMap = new HashMap<String,String>();

 public String encode(String str) {
  String encodedStr = encodedMap.get(str);
  if (encodedStr == null) {
   encodedStr = URLEncoder.encode(str);
   encodedMap.put(str, encodedStr);
  }
  return encodedStr;
 }
}
```

(The example is not thread save by purpose. We are not talking about concurrency, are we? ;) Also notice, that the one parameter encode method is deprecated now, because it uses system default encoding to encode the string.)

This cache would fill up the memory very quickly. It's never cleared after all. But there is also no special point in time when the cache should be cleared. The cached data never becomes outdated. Actually a cache should use a lot of memory if memory is not required by other subsystems, and only free the memory if it becomes required. For this purpose the Java runtime has the `SoftReference`, `WeakReference` and the utility classes which use them. `WeakHashMap` f.e. is the perfect match for this scenario. A `WeakHashMap` references the values using normal hard references, and the keys using weak references. As soon as the key is not referenced any more (soft or hard) the whole entry will be freed. Here's the example rewritten to use `WeakHashMap`:

```java
class CachedUrlEncoder {
 private static Map<String, String> encodedMap = new WeakHashMap<String,String>();

 public String encode(String str) {
  String encodedStr = encodedMap.get(str);
  if (encodedStr == null) {
   encodedStr = URLEncoder.encode(str);
   encodedMap.put(str, encodedStr);
  }
  return encodedStr;
 }
}
```

That was easy. Sadly you will notice at runtime that this code contains a memory leak. A not so obvious one. Let's analyse the situation.

As I mentioned already the map entries will be freed as soon as the key is not referenced by hard or soft references any more. In our case this should be immediately. After we've written the encoded string to the output stream of the JSP page, the string is not referenced any more. Still we're experiencing a memory leak. As so often the devil is in the details. The Sun implementation of `URLEncoder.encode()` tries to optimize by returning the reference to the string it received, if there is no encoding work to do. This is clever. It saves resources. But in this case this bit us really bad. If `encode()` returns the same reference the code will call `Map.put()` with the same reference as key and value. It'd look like:

```java
encodedMap.put(str, str);
```

After that line the map has an entry with a weak reference to `str` and a hard reference to `str`. The entry itself prevents that it is garbage collected!

That's mean.

The fix is simple, once one knows the cause. We render the optimization of `URLEncoder` useless:

```java
class CachedUrlEncoder {
 private static Map<String, String> encodedMap = new WeakHashMap<String,String>();

 public String encode(String str) {
  String encodedStr = encodedMap.get(str);
  if (encodedStr == null) {
   encodedStr = URLEncoder.encode(str);
   if (str == encodedStr) {
    encodedStr = new String(encodedStr);
   }
   encodedMap.put(str, encodedStr);
  }
  return encodedStr;
 }
}
```

Luckily the implementation of the string constructor is also smart. It does not copy the char data. A new string object is created and references the same char array as the old string. This is safe since strings are immutable. So the fix does create some overhead but not that much.
