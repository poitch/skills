---
name: wait-merge-pr
description: Wait for all PR checks to pass, then squash-merge and clean up.
disable-model-invocation: true
argument-hint: "[PR number, optional - auto-detects current branch PR]"
allowed-tools: Bash(gh *), Bash(git *), Bash(sleep *)
---

# Wait for Checks and Merge PR

Wait for all CI checks to pass on the current pull request, then squash-merge it and clean up.

## Step 1: Identify the PR

If the user provided a PR number as `$ARGUMENTS`, use that. Otherwise, detect the PR for the current branch:

```
gh pr view --json number,title,url,headRefName,state
```

If no PR is found or the PR is already merged/closed, inform the user and stop.

## Step 2: Wait for all checks to pass

Poll the PR's check status:

```
gh pr checks {number} --json name,state,conclusion
```

- If all checks have `state == "SUCCESS"` or `state == "SKIPPED"`, proceed to Step 3.
- If any check has `state == "FAILURE"`, inform the user which check(s) failed and stop. Do NOT merge.
- If any check is still `state == "PENDING"`, wait 30 seconds and poll again.
- Print a brief status update each time you poll (e.g. "Waiting for checks... (attempt 3/20) - 2/4 complete").
- After 20 attempts (10 minutes), inform the user the checks are still running and stop.

## Step 3: Merge the PR

Squash-merge the PR and delete the remote branch:

```
gh pr merge {number} --squash --delete-branch
```

If the merge fails (e.g. due to merge conflicts), inform the user and stop.

## Step 4: Switch to main and clean up

```
git checkout main && git pull
```

If the local branch still exists, delete it:

```
git branch -d {branch_name}
```

Ignore errors if the branch was already deleted.

## Step 5: Confirm

Let the user know:
- Which checks passed
- That the PR was merged
- That the branch is cleaned up and they are on main
