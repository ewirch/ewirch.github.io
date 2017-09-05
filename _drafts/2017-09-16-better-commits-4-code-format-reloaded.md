---
layout: article
title: "Better Commits - Part 4 - Code Format Reloaded"
categories: articles
date: 2017-09-16
modified: 2017-09-16
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

# Script
```bash
#/bin/bash

startRef=$1

# validate startRef
if !(git rev-parse --verify -q $startRef > /dev/null); then
	echo "ERRROR: Invalid ref."
	echo "Usage: formatCommit.sh COMMITISH-ID"
	echo " COMMITISH-ID Comit ID, ref name, branch name, etc..."
	echo ""
	exit 1
fi

origHead=$(git rev-parse HEAD)

## save current head
git update-ref -m "Before format script" HEAD HEAD

git reset -q --mixed $startRef >/dev/null 2>&1
if [ $? -ne 0 ]; then
	echo "'git reset' failed. See 'git reflog' to restore you original state."
	exit 1
fi

echo ""
git log $startRef..$origHead --pretty=format:'%Cred%h%Creset %Cblue<%an>%Creset
%C(yellow)%d%Creset %s' --abbrev-commit
echo ""
echo "Make sure the above commits are the ones you want to format. Switch to your IDE
and do so if the list is correct (press ENTER when finished). Simply press ENTER
immediately to cancel."
read

git reset -q --mixed $origHead >/dev/null 2>&1
if [ $? -ne 0 ]; then
echo "'git reset' failed. See 'git reflog' to restore you original state."
exit 1
fi

echo "You can verify and commit the format changes now."
```

# Usage
The script enables you to format all changed files between the current HEAD (inclusive) and the commit you specify on the command line (exclusive). This means you specify the last commit you don't want to format:

```
> git log
0004 my commit 4
0003 my commit 2
0002 my commit 1
0001 foreign commit
> formatCommits.sh 0001
```

The script resets the branch HEAD to the specified commit without changing the working copy. This way all committed changes appear as 'new' changes. Your IDE will show you all changed and new files. Format them in your IDE. Using this approach makes deleted files appear again (as deleted). Simply ignore them while formatting. When you're finished press ENTER in the console. The script will reset the branch HEAD back to the original head. All committed changes appear committed again. Only the format changes appear as changes. You can commit them separately now.

This time I made the script skip the commit step. I noticed already formatter bugs and some formatter behavior I didn't want to have in my code. It is good practice to check all changes introduced by the formatter before committing. You can revert format changes you don't like.

# Recovery
If anything goes wrong, you can get back to your original state at any time. The script saves the commit id of the original HEAD commit to the reflog before applying any changes. This is how it look like:

```
> git reflog
9ded58dbb5 HEAD@{0}: reset: moving to 9ded58dbb5a9501a866157cbe4fb61d7cf02b06c
42862a68e1 HEAD@{1}: reset: moving to 42862a68e1045ceec124be2703ca626c0e206405
9ded58dbb5 HEAD@{2}: Before format script
9ded58dbb5 HEAD@{3}: rebase -i (finish): returning to
refs/heads/feature/overridden-check
9ded58dbb5 HEAD@{4}: rebase -i (reword): Format
48cf53ae73 HEAD@{5}: rebase -i (reword): updating HEAD
```

The commit with the title "Before format script" has the id you are looking for: ```9ded58dbb5```. Reset your working copy to the original state to roll back:

```
> git reset --hard 9ded58dbb5
```