---
layout: article
title: "Better Commits - Part 2 - Refactorings"
categories: articles
date: 2017-08-31
modified: 2017-08-31
tags: [scm, git, code-review, code-history]
image:
  feature: 
  teaser: /2017/08/better-commits2-teaser.png
  path: /2017/08/better-commits2-teaser.png
  thumb: 
ads: false
comments: true
---


In the first part [Better Commits - Part 1 - Code Format]({{site.url}}/2017/08/better-commits-1-code-format.html) I presented a way how you can make your code history easier to read by committing format changes separately. This time I'll show how to handle refactorings. I will make heavy use of Git commands. If you use a different SCM or a Git UI, it's possible that you can map the steps to actions of your tool.

While working on a feature or bug fix I often find code which needs some love. Or I have to apply some refactorings to prepare the code for the new feature. It makes the commit history easier to read (and thus makes also code reviews easier) if the refactoring steps are committed separately. It reduces your cognitive overhead while parsing the changes. This way you group syntactic changes to commits.
Just imagine a rename of a highly used class. The change will affect a lot of files and a lot of lines. But the change itself is simple. If you don't split your changes into commits, then the rename change is mixed with feature implementation. The reviewer will have a hard time deciding line by line if the change is fine. If you commit the rename separately, the reviewer can quickly skim over the rename commit and concentrate on the logic change commit. Or if you get back to this commit in a couple of months, trying to find out what happened, you will have to check a lot of irrelevant lines to find the important changes.

But - Murphy knows - you only find code which needs refactoring after you already started making logic changes. You touched already a lot of files. How can you commit the refactoring separately?

# Stage/Fixup

If it's a simple refactoring -- so you can easily tell which lines are affected by the refactoring -- then you can select the hunks to commit manually. In Git language: you're staging (or adding to index) the changes. Go ahead and [read](https://git-scm.com/book/en/v2/Getting-Started-Git-Basics#_the_three_states) a [bit about](https://githowto.com/staging_changes) staging [changes in Git](https://softwareengineering.stackexchange.com/questions/119782/what-does-stage-mean-in-git), if you are not familiar with that topic yet.

This way your logic change remains uncommitted in the working copy, while the refactoring change is committed away. If you find the next refactoring while progressing, repeat the process. But if you notice something which belongs to the refactoring you already committed, you should commit a "fix-up". You do this by committing the change with a commit message which starts with the string "fixup!" and the first line of the commit message the change belongs to. For example, if you committed a refactoring with the commit message "small refactorings", then the new commit message should read "fixup! small refactorings". This even works if you committed other changes in between. Like this:

```text
87517e9 small refactoring
5e2eca7 some real changes
147e690 fixup! small refactoring
```

Why do we do this? Git has a powerful feature called [rebasing](https://git-scm.com/book/de/v1/Git-Branching-Rebasing). It allows you to change or rearrange commits. Using this commit message syntax and a command line flag, Git will automatically reorder the commits for us. To do this, you start a [interactive rebase](https://git-scm.com/book/id/v2/Git-Tools-Rewriting-History) and pass the parameter `--autosquash`:

```text
> git rebase -i --autosquash 87517e9^
```

(When rebasing you pass the id of the last commit which should not be changed by the rebase. I used "^" in the command above to tell Git to use the commit before `87517e9`.) Git will open a editor with contents similar to:

```text
pick 87517e9 small refactoring
fixup 147e690 fixup! small refactoring
pick 5e2eca7 some real changes
```

Git moved the fixup commit next to the commit it belongs and changed the action for the commit to "fixup". This will squash the commits and discard the commit message of the fixup commit.

It's possible you will get merge conflicts when moving commits. This happens when a change is based on a different commit which you crossed while moving. There are two ways to handle this. Either you resolve the merge conflicts manually. Or you don't squash the refactoring commits. The first approach is more clean and easier to read. The second approach is more pragmatic and easier to execute.


# Stash/Merge
What if the refactoring changes are so extended, that it would be too hard to pick all of them by hand? Then you can use the Git Stash. The Git Stash is a area where you can move changes to and get them back later.

You could stash your logic changes first. After you stashed your changes, the working copy will be clean (unchanged). You would apply the refactoring change, commit it and then get the stashed changes back. This is easy and quickly done. But it only works if your previous changes and the refactoring change do not intersect. If they do, then stashing the changes will get back the old code for you. Refactoring would be based on the old code. And un-stashing the changes will result in merge conflicts, because the stash was based on a older code base. I'll show you a approach here which works in all cases (even when your uncommitted changes and the refactoring change intersect).

With your uncommitted logic changes in your working tree, go ahead and apply your refactoring. Then stash the changes:

```text
> git stash --include-untracked
```

`--include-untracked` will also stash files you've created but have not yet committed or added to stage area. Your working copy is clean again. Without your logic changes and without the refactoring. Now apply your refactoring again. Usually you will use IDE tools for refactoring, which makes this step easy to repeat. Commit your refactoring. Now get back your changes from the stash:

```text
> git stash branch some-temporary-branch-name
> git symbolic-ref HEAD refs/heads/BRANCH
> git branch -D some-temporary-branch-name
```

We're not using `git stash pop`  here to avoid merge conflicts. We know the stashed code and the current branch contain the same refactoring. We can pick the state from the stash and ignore all conflicts. We do this by checking out the stashed state. We can't do simply `git checkout stash` because the stash command does some black magic to keep untracked files, staged files and changed files separate. We have to convert the stash to a regular branch first. You have to pick some unused temporary branch name. This get's you back your changes, including the refactoring. But it also switches your working copy to the temporary branch. Using the `symbolic-ref` command we tell Git to pretend that we're on our previous branch (you have to replace `BRANCH` in the command by your actual branch name), but without modifications to the working copy. Thus you end up on the refactoring commit, with your uncommitted changes before the stash, on top. The refactoring itself does not appear as a change any more. As a final step we clean up the temporary branch.

Voilà, we moved the refactoring to a separate commit without loosing our uncommitted changes.
It is important that you perform exactly the same refactoring in the second step which you've done earlier, and not more. If you do extra work in this step, it will be overwritten by the merge.


# Conclusion

I've shown two ways here which make it easier to group changes semantically. This makes code reviews and reading the history easier. Its a little bit more effort compared to committing everything in one change, but it pays off in my opinion.

