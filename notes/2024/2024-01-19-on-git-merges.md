# On git merges #git #branch #mess

We have repository A, and it's fork repository B. Usually weekly we synchronize changes from A to B. After big linting changes in A I forgot to switch from squash mode in git to merge without fast forward. In a result new merge show the same diff log (based on commits) and even giving merge conflict.
So we want to revert it. We agree on mess on history, so no `git push --force`.

```bash
git checkout -b fix-lint-merge-1
git log --oneline
# -m 1 shows we revert merge commit. Important flag
git revert -m 1 741c51d4
git fetch A
git merge remotes/A/master
```

In our case we have some unique changes in B. And as we did linting, so we can follow `ours`  strategy option.
```bash
git merge remotes/A/master --strategy-option ours
```

