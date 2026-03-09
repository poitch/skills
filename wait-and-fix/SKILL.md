---
name: wait-and-fix
description: Wait for Copilot code review to complete, then address any review comments.
disable-model-invocation: true
argument-hint: "[PR number, optional - auto-detects current branch PR]"
allowed-tools: Bash(gh *), Read, Edit, Glob, Grep, Skill
---

# Wait for Copilot Review and Fix Comments

Wait for the "Copilot code review" check to complete on a pull request, then address all review comments.

## Step 1: Identify the PR

If the user provided a PR number as `$ARGUMENTS`, use that. Otherwise, detect the PR for the current branch:

```
gh pr view --json number,title,url,headRefName
```

If no PR is found, inform the user and stop.

## Step 2: Wait for Copilot code review to complete

First, get the Copilot code review workflow ID:

```
gh api repos/{owner}/{repo}/actions/workflows --jq '.workflows[] | select(.name == "Copilot code review") | .id'
```

If no workflow is found, inform the user that no Copilot code review workflow was detected and stop.

Then poll the most recent run for that workflow on the PR's head branch:

```
gh api repos/{owner}/{repo}/actions/workflows/{workflow_id}/runs --jq '.workflow_runs[] | select(.head_branch == "refs/pull/{number}/head") | {id, status, conclusion}' | head -1
```

- If `status` is `completed`, proceed to Step 3.
- If `status` is `in_progress` or `queued`, wait 60 seconds and poll again.
- Print a brief status update each time you poll (e.g. "Waiting for Copilot code review... (attempt 3/10)").
- After 10 attempts (10 minutes), inform the user the check is still running and stop.

## Step 3: Address review comments

Once the Copilot check has completed, invoke the pr-comments skill to address the comments:

```
/pr-comments {number}
```
