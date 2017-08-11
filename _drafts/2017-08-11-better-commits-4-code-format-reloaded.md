---
layout: article
title: "Better Commits - Code Format"
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

In [Better Commits - Part 1 - Code Format]({{site.url}}/2017/08/better-commits-1-code-format.html) I introduced a script which commits the code format changes separately from the logic changes. Consequently committing the format separately makes code easier to review, the history easier to read and even easier rebasing/merging (in case of conflicts due to format changes). Sadly the process introduced by the script had it's own problems:
- The script was pretty complex and thus error prone.
- The script was not able to handle some situations.
- Format-Commits between multiple logic change commits make reordering / restructuring the commits hard.
- The script committed missing code format before the logic change. Effectively the source was updated to the current format rules before the logic change. If the logic change introduced format changes (line break, because of some inserted chars), this was committed together with the logic change.
- The script formatted only files touched in one commit. You had to remember to execute it for each logic change commit. Thus resulting in many distracting format commits.

So I came up with a new process and a new script. The new process uses post-change-format-commits. It commits the format **after** all commits of a change-set. The script respects all files in a change-set.

Until now I pressed CTRL-S now and then to update the source format (CTRL-S triggers a macro in my IntelliJ IDEA configuration, which does some clean up and re-formats the code). This leads to format changes to be merged together with logic changes.

Before logic change:

```java
void method(final int param1, final int pram2, final int param3, final int param4,
final int param5, final int param6, final int param7, final int param8) {
}
```

After logic change and format update:

```java
void method(final int param1,
             final int param2,
             final int param3,
             final int param4,
             final int param5,
             final int param6,
             final int param7,
             final int param8,
             final int param9) {
}
```

Suddenly 9 lines changed because of one new parameter. The new process requires you to change your behavior while coding. 

![one does not simly format tho whole source file]({{site.url}}/images/2017/08/one-does-not-simply.jpg)

You simply don't reformat the code or format just single lines or blocks if it's appropriate (CTRL-ALT-L in IntelliJ IDEA while you have the lines selected which should be formatted). Small changes to existing lines remain mostly unformatted. The change from the example above would lead to this code:

```java
void method(final int param1, final int pram2, final int param3, final int param4,
final int param5, final int param6, final int param7, final int param8,
             final int param9) {
}
```

IntelliJ does some basic formatting automatically while you type (it inserted a line break here).

This way the commit contains mainly logic changes. You run the format script after completing your change set (after all logic change commits). It will reformat all the changed source files completely and commit the changes separately (containing only format changes). If you'd like to restructure your change-set, you remove the last format commit (the format commit should always be last in your change-set), move / change the commits as you like and re-run the format script to add the format commit again. 