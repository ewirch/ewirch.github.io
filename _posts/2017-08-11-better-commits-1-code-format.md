---
title: "Better Commits - Part 1 - Code Format"
categories: articles
date: 2017-08-11
modified: 2017-08-11
tags: [scm, git, code-review, code-history]
image:
  feature: 
  teaser: /assets/images/2017/08/better-commits1-teaser.png
  path: /assets/images/2017/08/better-commits1-teaser.png
  height: 130
  thumb: 
---

Does anyone see the real change?

![diff with format]({{site.url}}/assets/images/2017/08/better-commits1-format-diff.png)

I spent many years reading source history and reviewing changes of others. I find that we often make it unnecessarily hard to read the changes. Important changes are mixed with less important, maybe unrelated changes. Multiple logic changes are committed together. Logic changes are mixed with refactorings. Commits split by files. Commits undoing changes done few commits before. I often find myself digging the commit history to find out why a particular change was made. A unclear commit history makes this hard and costs productivity. Remember the WORM principle: write once, read many. You will read the code over and over again in the future. Also when reviewing a change (pull request), it's hard to review unclean commit history. Modern review tools allow to review the changes separated by commits. If the author invested some effort in nice change splitting, the review process is faster, more efficient and maybe more profitable. Many unimportant changes wear down the reviewer's attention. He might not notice problematic changes. Or maybe to tired, after reviewing 30 classes, for a good improvement idea.

I'd like to present some ideas and techniques which will improve your commit history. This post is part of a multi post series.

- [Better Commits - Part 1 - Code Format][]
- [Better Commits - Part 2 - Refactorings][]
- [Better Commits - Part 3 - Review Changes][]
- [Better Commits - Part 4 - Code Format Reloaded][]


The diff above contains a lot of noise. It creates too much cognitive overhead while parsing. I often have to review changes like this. Code formatting rules which are used by the whole team are important. But it still happens. Maybe the committer forgot to format a file after he applied changes. Or maybe the formatting rules were changed after the file was creating. Touching the file again re-formats it completely.

How about you review this change instead?

![diff with format]({{site.url}}/assets/images/2017/08/better-commits1-no-format-diff.png)

Now we see the important stuff.

Using Git and a automatic formatter it's easy to separate logic changes from format changes. I created a script which automates this process. The script commits the missing format before your change. It fixes the format-rules-changed and the someone-forgot-to-format problems. For example: you changed the files `A.java`, `B.java` and `C.java`. Only `C.java` had the correct format before you changed it. The script will get the format of `A.java` and `B.java` back on the line. And commit it.

```bash
#!/bin/bash

# collect list of changed files, ignore added files, filtered to .java files
FILES=`git status --porcelain | awk 'match($1, "M"){print $2}' | grep "\.java$"`

# if change set empty, exit
if [ -z "$FILES" ]; then
	echo "No changes. Exiting."
	exit 0
fi

# stash changes
git stash

# modify changed files
tmpfile="oeuioetnhaoestudaoesudaoeuasasoe624942947954.txt"
for FILE in $FILES; do
	echo " " > $tmpfile
	cat $FILE >> $tmpfile
	mv $tmpfile $FILE
done

# pasue, allow to format code
echo
echo
echo "Format modified files in IDE. Hit enter when you're done."
read

# commit changes "Fromatting"
git add $FILES
git commit -m "Formatting"

# unstash changes, merge strategy = theirs
git cherry-pick -n -m1 -Xtheirs stash

git stash drop stash@{0}
```

# Usage

1. Change the code as you normally would do.
1. Before you commit your logic changes, run the script.
1. The script stashes all your changes and touches the files to make them appear as changed again.
1. The script pauses and waits for you. Change into your IDE and format all changed files. In IntelliJ IDEA, open the Version Control window (ALT-9) and switch to Local Changes tab. Select all files you want to format and press CTRL-ALT-L (Reformat Code).
1. Now press ENTER in console to continue the script.
1. The script commits the changes and restores the original changes.
1. You can commit your changes now. Don't forget to format. ;)


[Better Commits - Part 1 - Code Format]: {{site.url}}{% post_url 2017-08-11-better-commits-1-code-format %}
[Better Commits - Part 2 - Refactorings]: {{site.url}}{% post_url 2017-08-31-better-commits-2-refactorings %}
[Better Commits - Part 3 - Review Changes]: {{site.url}}{% post_url 2017-09-09-better-commits-3-review-changes %}
[Better Commits - Part 4 - Code Format Reloaded]: {{site.url}}{% post_url 2017-09-16-better-commits-4-code-format-reloaded %}
