---
title: "Better Commits - Part 3 - Review Changes"
categories: articles
date: 2017-09-09
modified: 2017-09-09
tags: [scm, git, code-review, code-history]
image:
  feature: 
  teaser: /assets/images/2017/09/better-commits3-teaser.jpg
  path: /assets/images/2017/09/better-commits3-teaser.jpg
  thumb: 
---

In our team we're doing code reviews. But they tend to render the commit history messy.

This post is part of a multi post series.

- [Better Commits - Part 1 - Code Format][]
- [Better Commits - Part 2 - Refactorings][]
- [Better Commits - Part 3 - Review Changes][]
- [Better Commits - Part 4 - Code Format Reloaded][]

Don't get me wrong. I love code reviews. If you don't do code reviews, you should start today. Code reviews increase the chance of finding bugs before they get merged to master / dev. Code reviews improve class design. The reviewer has to understand the change. If the reviewer does'n understand it, then the design is obviously bad and should be improved. The reviewer can ask for clarification and the [code will be changed][] to clarify the topic for future readers. You can learn a lot from review remarks. The reviewer can remind you to stick with your team code guidelines.

The review process sometimes involves multiple rounds of remarks and changes. How are those changes committed?

#### New Commits

Usually the author will add new commits, which address the review remarks. This can end up like this (oldest commit at the bottom):

```text
88dab2314d Review remarks
52ad268a3f Review remarks
348ee42a2b Review remarks
13d35e2c46 Implement ForwardToRenderer
87ba9c547f Extract methods for readability
3464db3dde Rename RenderingService
```
This is a history of change. From this point of view the log is "correct". But it's not helpful at all, if you read the commit log later. It is not a useful information to know if the change was part of the first, second or third review round. This approach has only one important benefit: the reviewer can concentrate on few commits in the next review round.

Instead of creating new history from review remarks, the changes should be incorporated in to the existing commits, or be split in a semantic way, if they don't belong to the previous commits.


#### Rebase

One would use the [interactive Git rebase command][] to do this. [Rebase][] allows you to rewrite the commit history. This way the code history would look like the code has	 been written like this from the beginning. The reader is not bothered with the review process. But: rebasing creates completely new commits. Your review tool has no chance to detect that the newly pushed commits have been review already. It will show them as new commits. Everything appears as not reviewed again. The reviewer will have to check all changes again to make sure that nothing broke.


#### The Solution: Fixup Commits

Rebase was a step in the correct direction. But instead of rebasing after each review round, the author should commit fixup commits and rebase only once at the end of the review. Fixup commits are intended to be merged with previous commits later. You can do this manually: name them as you like and reorder them when rebasing. But you could also use Git features to aid you in this. If you follow a commit message format, Git will automatically place the fixup commit on the right position in the interactive rebase step.

Let's start with the commit example from above:

```text
13d35e2c46 Implement ForwardToRenderer
87ba9c547f Extract methods for readability
3464db3dde Rename RenderingService
```
The reviewer didn't like the name of `ForwardToRenderer`. You rename the class and use this commit message to commit it:

```text
fixup! Implement ForwardToRenderer

renamed ForwardToRenderer to RendererForwarder.
```
The magic word is the first "fixup!" [followed by the first line of the commit][] it should be merged with. Simply use the option `--autosquash` when invoking interactive rebase. Fixup commit messages will be discarded during rebase, so you can use the commit message body to leave some clarifications for the reviewer about what the commit contains.

Using this approach the reviewer only has to review changes applied after his remarks. And at the same time the commit history will be fixed after the review.

[Better Commits - Part 1 - Code Format]: {{site.url}}{% post_url 2017-08-11-better-commits-1-code-format %}
[Better Commits - Part 2 - Refactorings]: {{site.url}}{% post_url 2017-08-31-better-commits-2-refactorings %}
[Better Commits - Part 3 - Review Changes]: {{site.url}}{% post_url 2017-09-09-better-commits-3-review-changes %}
[Better Commits - Part 4 - Code Format Reloaded]: {{site.url}}{% post_url 2017-09-16-better-commits-4-code-format-reloaded %}
[code will be changed]: https://testing.googleblog.com/2017/06/code-health-too-many-comments-on-your.html
[interactive Git rebase command]: https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History
[Rebase]: https://git-scm.com/book/en/v2/Git-Branching-Rebasing
[followed by the first line of the commit]: https://git-scm.com/docs/git-rebase#git-rebase---autosquash
