---
name: create-pr
description: Create a new branch from main, commit all changes, push, and open a pull request.
disable-model-invocation: true
argument-hint: "[branch name, optional - auto-generates from changes]"
allowed-tools: Bash(git *), Bash(gh *)
---

# Create PR

Create a new branch, commit all current changes, push, and open a pull request.

## Step 1: Check for changes

```
git status
```

If there are no staged or unstaged changes and no untracked files, inform the user there is nothing to commit and stop.

## Step 2: Review changes

Run these in parallel to understand what will be committed:

```
git diff
git diff --cached
git log --oneline -5
```

## Step 3: Create a new branch

If the user provided a branch name as `$ARGUMENTS`, use that. Otherwise, analyze the changes and generate a short, descriptive kebab-case branch name (e.g. `add-cron-tools`, `fix-auth-redirect`).

Ensure you are branching from the latest main:

```
git checkout main && git pull && git checkout -b {branch_name}
```

If already on a non-main branch and there are uncommitted changes, stash first, switch, then unstash:

```
git stash && git checkout main && git pull && git checkout -b {branch_name} && git stash pop
```

## Step 4: Stage all changes

```
git add -A
```

## Step 5: Generate commit message

Analyze the staged diff to write a concise commit message:
- First line: imperative summary under 72 characters (e.g. "Add cron job scheduling tools")
- If needed, add a blank line followed by a short body explaining the "why"
- Follow the style of recent commits from Step 2

## Step 6: Commit

Create the commit. Always use a HEREDOC to pass the message:

```
git commit -m "$(cat <<'EOF'
<commit message here>

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

If the commit fails (e.g. pre-commit hook), inform the user and stop. Do NOT retry with `--no-verify`.

## Step 7: Push

```
git push -u origin {branch_name}
```

If the push fails, inform the user and stop. Do NOT force push.

## Step 8: Create pull request

Create the PR using `gh`. Use a HEREDOC for the body:

```
gh pr create --title "<short title under 72 chars>" --body "$(cat <<'EOF'
## Summary
<1-3 bullet points describing the changes>

## Test plan
<bulleted checklist of how to verify the changes>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

The PR title should match or closely follow the commit message first line.

## Step 9: Confirm

Show the user:
- The branch name
- The commit hash and message
- The PR URL
