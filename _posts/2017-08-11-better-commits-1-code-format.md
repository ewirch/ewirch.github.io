---
layout: article
title: "Better Commits - Part 1 - Code Format"
categories: articles
date: 2017-08-11
modified: 2017-08-11
tags: [scm, git, code-review, code-history]
image:
  feature: 
  teaser: /2017/08/better-commits1-teaser.png
  path: /2017/08/better-commits1-teaser.png
  hight: 130
  thumb: 
ads: false
comments: true
---

Does anyone see the real change?

![diff with format]({{site.url}}/images/2017/08/better-commits1-format-diff.png)

The diff contains a lot of noise. It creates too much cognitive overhead while parsing. I often have to review changes like this. Code formatting rules which are used by the whole team are important. But it still happens. Maybe the committer forgot to format a file after he applied changes. Or maybe the formatting rules were changed after the file was creating. Touching the file again re-formats it completely.

How about you review this change instead?

![diff with format]({{site.url}}/images/2017/08/better-commits1-no-format-diff.png)

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

