---
name: commit-and-push
description: Stage all changes, commit with an auto-generated message, and push to the remote.
disable-model-invocation: true
argument-hint: "[commit message, optional - auto-generates from diff]"
allowed-tools: Bash(git *), Bash(ufmt *)
---

# Commit and Push

Stage all current changes, create a commit, and push to the remote branch.

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

## Step 3: Format changed Python files

Get the list of modified and new Python files, then activate the venv and run `ufmt format` on them:

```
source venv/bin/activate && git diff --name-only --diff-filter=ACMR HEAD -- '*.py' | xargs -r ufmt format
```

If no Python files were changed, skip this step.

## Step 4: Stage all changes

```
git add -A
```

## Step 5: Generate commit message

If the user provided a message as `$ARGUMENTS`, use that exactly.

Otherwise, analyze the staged diff to write a concise commit message:
- First line: imperative summary under 72 characters (e.g. "Add memory tool validation")
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

Push to the current branch's remote tracking branch:

```
git push
```

If there is no upstream branch yet, use:

```
git push -u origin HEAD
```

If the push fails (e.g. rejected due to remote changes), inform the user and stop. Do NOT force push.

## Step 8: Confirm

Show the user:
- The commit hash and message
- Which branch was pushed and to where
