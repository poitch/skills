# Agent Skills

A collection of reusable skills for AI coding agents that automate common Git and GitHub PR workflows. Install them into your agent to streamline pull request management — from creating PRs to waiting on reviews and addressing comments automatically.

## Skills

| Skill | Description |
|-------|-------------|
| `/commit-and-push` | Stage all changes, generate a commit message from the diff, and push to the remote |
| `/create-pr` | Create a branch from main, commit changes, push, and open a pull request |
| `/merge-pr` | Squash-merge a PR, delete the branch, and switch back to main |
| `/pr-comments` | Fetch unresolved PR review comments and address each one with code changes |
| `/wait-and-fix` | Wait for Copilot code review to finish, then auto-fix all review comments |
| `/wait-merge-pr` | Wait for all PR checks to pass, then squash-merge and clean up |
| `/land-pr` | Take a PR from draft to merged: mark ready, get it reviewed (CodeRabbit, falling back to Durian when CodeRabbit is throttled), address the review, and squash-merge |

## Installation

Install with the [`skills`](https://github.com/vercel-labs/skills) CLI:

```sh
npx skills add poitch/skills
```

Choose Claude Code (and any other agents you use) when prompted, and install
globally to make the skills available in every project. The CLI copies each skill
into `~/.agents/skills/`, symlinks it into each agent's skills directory (for
Claude Code, `~/.claude/skills/`), and records the source in
`~/.agents/.skill-lock.json` so they can be updated later.

Each skill is a self-contained directory with a `SKILL.md` that the agent follows step-by-step. No additional dependencies required beyond `git` and `gh` (GitHub CLI).

`/land-pr` assumes a repository reviewed by CodeRabbit, with a Durian review bot
(`@durian-review`) as the fallback reviewer when CodeRabbit is rate-limited.

## How It Works

Skills are declarative workflows written in Markdown. When invoked, the agent reads the `SKILL.md` and executes each step — running shell commands, reading files, and making edits as needed. Skills can compose with each other (e.g., `/wait-and-fix` calls `/pr-comments` after the review check completes).

## Contributing

Add a new directory with a `SKILL.md` file following the existing patterns. Each skill should have a YAML front matter block with `name`, `description`, and `allowed-tools`.
