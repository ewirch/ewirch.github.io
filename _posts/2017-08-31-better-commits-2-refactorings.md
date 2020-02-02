---
title: "Better Commits - Part 2 - Refactorings"
categories: articles
date: 2017-08-31
modified: 2017-09-30
tags: [scm, git, code-review, code-history]
image:
  feature: 
  teaser: /assets/images/2017/08/better-commits2-teaser.png
  path: /assets/images/2017/08/better-commits2-teaser.png
  thumb: 
---


In the first part [Better Commits - Part 1 - Code Format][] I presented a way how you can make your code history easier to read by committing format changes separately. This time I'll show how to handle refactorings. I will make heavy use of Git commands. If you use a different SCM or a Git UI, it's possible that you can map the steps to actions of your tool.

This post is part of a multi post series.

- [Better Commits - Part 1 - Code Format][]
- [Better Commits - Part 2 - Refactorings][]
- [Better Commits - Part 3 - Review Changes][]
- [Better Commits - Part 4 - Code Format Reloaded][]

While working on a feature or bug fix I often find code which needs some love. Or I have to apply some refactorings to prepare the code for the new feature. It makes the commit history easier to read (and thus makes also code reviews easier) if the refactoring steps are committed separately. It reduces your cognitive overhead while parsing the changes. This way you group syntactic changes to commits.
Just imagine a rename of a highly used class. The change will affect a lot of files and a lot of lines. But the change itself is simple. If you don't split your changes into commits, then the rename change is mixed with feature implementation. The reviewer will have a hard time deciding line by line if the change is fine. If you commit the rename separately, the reviewer can quickly skim over the rename commit and concentrate on the logic change commit. Or if you get back to this commit in a couple of months, trying to find out what happened, you will have to check a lot of irrelevant lines to find the important changes.

But - Murphy knows - you only find code which needs refactoring after you already started making logic changes. You touched already a lot of files. How can you commit the refactoring separately?

# Stage/Fixup

If it's a simple refactoring -- so you can easily tell which lines are affected by the refactoring -- then you can select the hunks to commit manually. In Git language: you're staging (or adding to index) the changes. Go ahead and [read][] a [bit about][] staging [changes in Git][], if you are not familiar with that topic yet.

This way your logic change remains uncommitted in the working copy, while the refactoring change is committed away. If you find the next refactoring while progressing, repeat the process. But if you notice something which belongs to the refactoring you already committed, you should commit a "fix-up". You do this by committing the change with a commit message which starts with the string "fixup!" and the first line of the commit message the change belongs to. For example, if you committed a refactoring with the commit message "small refactorings", then the new commit message should read "fixup! small refactorings". This even works if you committed other changes in between. Like this:

```text
87517e9 small refactoring
5e2eca7 some real changes
147e690 fixup! small refactoring
```

Why do we do this? Git has a powerful feature called [rebasing][]. It allows you to change or rearrange commits. Using this commit message syntax and a command line flag, Git will automatically reorder the commits for us. To do this, you start a [interactive rebase][] and pass the parameter `--autosquash`:

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

# Fixing The Rename Case

The above approach will yield awkward results if a file rename is part of the refactoring. This is because the renamed file is added to the index without your local changes (if you use `git mv` or your IDE for renaming). Stashing will also stash the index. Then, performing the refactoring will change the file contents. But restoring the stash will also restore the index with an old version of the file.

I've written a Python script, which performs the Git magic to perform the refactoring and also handles the rename situation: `commitRefactoring.py`

```python
#!/usr/bin/python
from subprocess import run, PIPE, CompletedProcess
from sys import stderr

DEBUG = False
commandList = []


def run_checked(command: [str], input_string=None) -> CompletedProcess:
    add_to_command_list(command)
    result = run(command, input=input_string, stdout=PIPE, stderr=PIPE, encoding="UTF-8")
    if result.returncode > 1:
        print_command_fail_stack(result)
    return result


def add_to_command_list(command):
    command_string = " ".join(command)
    commandList.append(command_string)
    if DEBUG:
        print(command_string)


def print_command_fail_stack(result):
    print("Previous commands:", file=stderr)
    for command_string in commandList:
        print("   " + command_string, file=stderr)
    print(result.stdout, file=stderr)
    print(result.stderr, file=stderr)
    print("""
Get back your work:
 $ git reflog
Locate the "refactoring scrip backup line".
 $ git reset --hard abcdef
 $ git clean -f
 $ git stash apply --index
""")
    exit(1)


def create_unique_branch_name() -> str:
    counter = 0
    branch_name, result = verify_branch_name(counter)
    while not result.returncode:
        counter += 1
        branch_name, result = verify_branch_name(counter)
    return branch_name


def verify_branch_name(counter):
    branch_name = "tmp-branch-" + str(counter)
    result = run_checked(["git", "rev-parse", "--verify", branch_name])
    return branch_name, result


def verify_worktree_clean() -> bool:
    # Check if worktree is clean:
    #  dirty working tree or uncommitted changes:
    #   git diff-index --quiet HEAD --
    #    0 = no uncommitted changes
    #    1 = uncommitted changes
    #    >1 = error
    result = run_checked(["git", "diff-index", "--quiet", "HEAD"])
    if result.returncode == 1:
        return False
    # check for un-tracked un-ignored files, they will cause the un-stash to fail (if the stash contains them too)
    #   git ls-files --exclude-standard --others --error-unmatch -- .
    #    0 = un-tracked, un-ignored files
    #    1 = clean
    #    >1 = error
    result = run_checked(["git", "ls-files", "--exclude-standard", "--others", "--error-unmatch", "--", "."])
    if result.returncode == 0:
        return False
    return True


class StatusLine:
    def __init__(self):
        self.lineId = None
        self.status = None
        self.fileModeHead = None
        self.fileModeIndex = None
        self.fileModeWorkTree = None
        self.hashHead = None
        self.hashIndex = None
        self.path = None
        self.originalPath = None


def git_status() -> [StatusLine]:
    result = run_checked(['git', 'status', '--porcelain=2'])
    stats = []
    for line in result.stdout.splitlines():
        if line[0] == '1':
            # modified entries
            status = StatusLine()
            (
                status.lineId, status.status, status.subModuleState, status.fileModeHead, status.fileModeIndex,
                status.fileModeWorkTree, status.hashHead, status.hashIndex, status.path) = line.split(sep=" ")
            stats.append(status)
        if line[0] == '2':
            # renamed entries
            status = StatusLine()
            (
                status.lineId, status.status, status.subModuleState, status.fileModeHead, status.fileModeIndex,
                status.fileModeWorkTree, status.hashHead, status.hashIndex, status.renameScore, paths) = line.split(
                sep=" ")
            (status.path, status.originalPath) = paths.split(sep="\t")
            stats.append(status)
    return stats


def get_current_branch_name() -> str:
    result = run_checked(["git", "symbolic-ref", "HEAD"])
    # stdout contains line breaks. splitlines removes them.
    return result.stdout.splitlines()[0] if result else ""


def stash_changes():
    run_checked(['git', 'update-ref', '-m', "refactoring script backup", "HEAD", "HEAD"])
    run_checked(["git", "stash", "save", "-q", "--include-untracked", "refactoring script backup"])
    run_checked(["git", "stash", "apply", "-q", "--index"])
    run_checked(["git", "stash", "save", "-q", "--include-untracked"])


def collect_deleted_files():
    deleted = []
    for status in git_status():
        if status.status[1] == 'D':
            # each deleted entry
            deleted.append(status.path)
    return deleted


def unstash_changes(previous_branch: str):
    unique_branch_name = create_unique_branch_name()
    run_checked(["git", "stash", "branch", unique_branch_name])
    run_checked(["git", "symbolic-ref", "HEAD", previous_branch])
    run_checked(["git", "branch", "-d", unique_branch_name])


def update_refactored_files(before_states):
    after_states = git_status()
    for beforeState in before_states:
        for afterState in after_states:
            if beforeState.path == afterState.path and beforeState.hashHead != afterState.hashHead:
                # replace the indexed hash by the new head hash
                index_entry = f"{afterState.fileModeIndex} {afterState.hashHead} 0\t{afterState.path}"
                run_checked(["git", "update-index", "--index-info"], index_entry)


def detached_head():
    result = run_checked(["git", "symbolic-ref", "-q", "HEAD"])
    return result.returncode == 1


def main():
    if detached_head():
        print("""
You are in 'detached HEAD' state. This script uses branches to do it's work. Check out the branch you want to work on. 
""")
        exit(1)
    current_branch = get_current_branch_name()
    pre_change_state = git_status()
    stash_changes()

    input("Apply refactoring now and commit it. Press ENTER when finished.")

    while not verify_worktree_clean():
        print("""
Work tree is dirty. Did you forget to commit your refactoring?
If the work tree is not clean, unstashing of the previous changes might fail.
Commit or revert your changes, remove any untracked un-ignored files and press ENTER when finished.
""")
        input()

    unstash_changes(current_branch)
    update_refactored_files(pre_change_state)


main()
```


# Conclusion

I've shown two ways here which make it easier to group changes semantically. This makes code reviews and reading the history easier. Its a little bit more effort compared to committing everything in one change, but it pays off in my opinion.

[Better Commits - Part 1 - Code Format]: {{site.url}}{% post_url 2017-08-11-better-commits-1-code-format %}
[Better Commits - Part 2 - Refactorings]: {{site.url}}{% post_url 2017-08-31-better-commits-2-refactorings %}
[Better Commits - Part 3 - Review Changes]: {{site.url}}{% post_url 2017-09-09-better-commits-3-review-changes %}
[Better Commits - Part 4 - Code Format Reloaded]: {{site.url}}{% post_url 2017-09-16-better-commits-4-code-format-reloaded %}
[read]: https://git-scm.com/book/en/v2/Getting-Started-Git-Basics#_the_three_states
[bit about]: https://githowto.com/staging_changes
[changes in Git]: https://softwareengineering.stackexchange.com/questions/119782/what-does-stage-mean-in-git
[rebasing]: https://git-scm.com/book/de/v1/Git-Branching-Rebasing
[interactive rebase]: https://git-scm.com/book/id/v2/Git-Tools-Rewriting-History
