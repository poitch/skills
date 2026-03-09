---
name: merge-pr
description: Squash-merge the current PR, delete the branch, and switch back to main.
disable-model-invocation: true
argument-hint: "[PR number, optional - auto-detects current branch PR]"
allowed-tools: Bash(gh *), Bash(git *)
---

# Merge PR and Clean Up

Squash-merge the current pull request, delete the remote and local branch, and switch back to main.

## Step 1: Identify the PR

If the user provided a PR number as `$ARGUMENTS`, use that. Otherwise, detect the PR for the current branch:

```
gh pr view --json number,title,url,headRefName,state
```

If no PR is found or the PR is already merged/closed, inform the user and stop.

## Step 2: Merge the PR

Squash-merge the PR and delete the remote branch in one step:

```
gh pr merge {number} --squash --delete-branch
```

If the merge fails (e.g. due to merge conflicts or failing checks), inform the user and stop.

## Step 3: Switch to main

```
git checkout main && git pull
```

## Step 4: Clean up local branch

If the local branch still exists (it may already have been removed by `--delete-branch`), delete it:

```
git branch -d {branch_name}
```

Ignore errors if the branch was already deleted.

## Step 5: Confirm

Let the user know the PR was merged, the branch is cleaned up, and they are on main.
