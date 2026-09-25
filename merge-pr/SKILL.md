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

Squash-merge the PR with auto-merge enabled and delete the remote branch in one step:

```
gh pr merge {number} --squash --delete-branch --auto
```

`--auto` queues the merge to fire as soon as required checks go green and branch protection is satisfied. If checks are already green, GitHub merges immediately. This is the right default — it works regardless of whether checks are pending or already passing, and it doesn't require waiting on CI before invoking the skill.

If the merge fails (e.g. due to merge conflicts, an unmergeable state, or auto-merge being disabled at the repo level), inform the user and stop.

## Step 3: Confirm

If the merge happened immediately, GitHub returns success and `gh pr view --json state` will show `MERGED`. If auto-merge queued the merge, `gh pr view --json autoMergeRequest,state` will show the queue entry and `state: OPEN`.

If the PR is already merged (state `MERGED`), proceed to step 4. Otherwise, tell the user the merge is queued and stop here — the local cleanup steps below assume the merge actually happened so we can fast-forward main, and we don't want to leave them on a stale main while their PR is still pending.

## Step 4: Switch to main

Only run this when the PR has already merged:

```
git checkout main && git pull
```

## Step 5: Clean up local branch

If the local branch still exists (it may already have been removed by `--delete-branch`), delete it:

```
git branch -d {branch_name}
```

Ignore errors if the branch was already deleted.

## Step 6: Confirm

Let the user know the PR was merged, the branch is cleaned up, and they are on main.
