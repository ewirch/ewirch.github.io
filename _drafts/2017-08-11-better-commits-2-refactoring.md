---
layout: article
title: "Better Commits - Part 2 - Refactoring"
categories: articles
date: 2017-08-11
modified: 2017-08-11
tags: [scm, git, code-review, code-history]
image:
  feature: 
  teaser: /2013/11/its-not-my-code.jpg
  path: /2013/11/its-not-my-code.jpg
  thumb: 
ads: false
comments: true
---


In the first part [Better Commits - Part 1 - Code Format]({{site.url}}/2017/08/better-commits-1-code-format.html) I presented a way how you can make your code history easier to read by committing format changes separately. This time I'll show how to handle refactorings so your commit history will be easy to read.

While working on a feature or bug fix I often find code which needs some love. 