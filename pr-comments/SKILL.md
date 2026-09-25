---
name: pr-comments
description: Pull all file comments from the current GitHub pull request and address them by making code changes.
disable-model-invocation: true
argument-hint: "[PR number, optional - auto-detects current branch PR]"
allowed-tools: Bash(gh *), Read, Edit, Glob, Grep
---

# Address PR Comments

Pull all review comments from the current GitHub pull request and address each one by making the necessary code changes.

## Step 1: Identify the PR and check out its branch

If the user provided a PR number as `$ARGUMENTS`, use that. Otherwise, detect the PR for the current branch:

```
gh pr view [number] --json number,title,url,headRefName,state
```

If no PR is found (or it is already merged/closed), inform the user and stop.

**Then check out the PR's branch before editing anything.** When a PR number is
passed, you are often on a different branch (e.g. `main`), so edits would
otherwise land on the wrong branch — the changes never reach the PR and may
pollute `main`'s working tree. Switch to the PR's head branch first:

```
git fetch origin <headRefName>
git checkout <headRefName>
git pull --ff-only    # if it tracks a remote
```

If the working tree has uncommitted changes that block the checkout, STOP and
tell the user — do not stash, discard, or force-switch. Only start reading and
editing files once `git branch --show-current` confirms you are on
`<headRefName>`.

## Step 2: Fetch all review comments

Get all review comments (file-level comments, not general PR comments) using:

```
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate --jq '.[] | {id, path, line, original_line, side, body, diff_hunk, subject_type, user: .user.login, created_at, in_reply_to_id}'
```

Also fetch review threads via GraphQL to understand resolved vs unresolved status (note: `gh pr view --json` does NOT support `reviewThreads`):

```
gh api graphql -f query='
{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {number}) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          comments(first: 10) {
            nodes {
              databaseId
              path
              originalStartLine
              startLine
              body
              author { login }
            }
          }
        }
      }
    }
  }
}'
```

Filter the results to only unresolved threads (`isResolved == false`). The `id` field on each thread node is the GraphQL node ID needed to resolve the thread later.

## Step 3: Filter and organize

- **Only address unresolved comment threads** - skip any threads that are already resolved.
- **Skip reply comments** (those with `in_reply_to_id` set) - only process top-level comments in each thread.
- **Group comments by file path** for efficient processing.
- Present a summary to the user showing each comment with:
  - File path and line number
  - Who wrote the comment
  - The comment body
  - The relevant code context (from diff_hunk)

## Step 4: Address each comment

For each unresolved comment:

1. **Read the relevant file** to understand the full context around the commented line.
2. **Analyze the comment** to determine what change is being requested.
3. **Make the code change** using the Edit tool.
4. **Resolve the comment thread** on GitHub using the GraphQL API:
   ```
   gh api graphql -f query='mutation { resolveReviewThread(input: {threadId: "<thread_node_id>"}) { thread { isResolved } } }'
   ```
   To get the thread node ID, include `id` (the GraphQL node ID) when fetching review threads in Step 2.
5. **Briefly explain** what you changed and why.

If a comment is unclear or requires a design decision, flag it to the user instead of guessing. Do NOT resolve these threads.

## Step 5: Summary

After addressing all comments, provide a summary:
- List each comment and what was done to address it
- Note any comments that were skipped or need human input
- Remind the user to review the changes before committing
