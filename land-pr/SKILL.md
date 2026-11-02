---
name: land-pr
description: Take a GitHub PR through the full landing process — mark ready, run review (CodeRabbit, falling back to a configurable review bot only when CodeRabbit is throttled), monitor until the review completes, address the comments, and squash-merge.
disable-model-invocation: true
argument-hint: "[PR number, optional - auto-detects current branch PR]"
allowed-tools: Bash(gh *), Bash(git *), Bash(printenv *), Read, Edit, Glob, Grep
---

# Land a PR

Take a pull request from draft to merged: mark it ready, get it reviewed, address
the review, and merge it. Run the steps in order; do not skip ahead.

## Configuration: the fallback reviewer

CodeRabbit is the primary reviewer. When it is throttled, this skill can hand the
review to a second bot — but which bot, and how to summon it, is per-repository,
so it comes from environment variables rather than being written in here. Set
them in the `env` block of Claude Code's settings: the repository's committed
`.claude/settings.json` (travels with the repo — best when the bot is specific to
it) or your `~/.claude/settings.json` (applies everywhere).

| Variable | Required | Meaning |
|---|---|---|
| `LAND_PR_FALLBACK_COMMENT` | to enable the fallback | The exact comment that summons the bot, e.g. `@review-bot review` |
| `LAND_PR_FALLBACK_LOGIN` | no | Regex matching the bot's GitHub login. Default: the handle from the comment's leading `@mention` (`review-bot`) |
| `LAND_PR_FALLBACK_DONE` | no | Regex a comment from the bot matches once its review is finished, e.g. `Status.*REVIEWED`; that comment must also be about the current head (Step 3). Default: the bot has submitted a PR review on the head, or posted a comment after the summon |

```json
{
  "env": {
    "LAND_PR_FALLBACK_COMMENT": "@review-bot review",
    "LAND_PR_FALLBACK_DONE": "Status.*REVIEWED"
  }
}
```

Read them at the start (an unset variable prints nothing):

```
printenv LAND_PR_FALLBACK_COMMENT LAND_PR_FALLBACK_LOGIN LAND_PR_FALLBACK_DONE
```

With no `LAND_PR_FALLBACK_COMMENT`, there is **no fallback**: if CodeRabbit is
throttled, tell the user and stop — do not invent a reviewer to ping. Say that it
won't come back to this head by itself (see Step 2): it reviews again on the next
push, or when the user asks it to.

## CodeRabbit's result on the head

Steps 2, 3 and 5 all ask the same question: what has CodeRabbit concluded about
the PR's **current head commit**? Depending on how it is installed, CodeRabbit
answers on that commit either as a **commit status** or as a **check run** — a
repository can have one and never the other, so read both:

```
HEAD=$(gh pr view {number} --json headRefOid --jq .headRefOid)
gh api repos/{owner}/{repo}/commits/$HEAD/status \
  --jq '.statuses[]|select(.context|test("CodeRabbit";"i"))|"\(.state) \(.description)"'
gh api repos/{owner}/{repo}/commits/$HEAD/check-runs \
  --jq '[.check_runs[]|select(.name|test("coderabbit";"i"))]|sort_by(.id)|last
        |"\(.status) \(.conclusion) \(.output.summary // "" | split("\n")[0])"'
```

The status's description, or the latest check run's summary (a commit can have
several runs — the draft-time one and later ones; the highest `id` is the latest),
says one of:

- ***Review completed*** — CodeRabbit reviewed this head.
- ***Review in progress*** — it started; wait.
- ***Review skipped*** — usually the draft-time result (see Step 2), not a verdict.
- ***Rate limited*** — throttled on this head (see Step 2).

Nothing at all on the head means CodeRabbit hasn't processed it yet. Never read
this from the PR's comments, its review objects, or `gh pr checks` — Step 3 says
why each one misleads.

## Step 1: Identify the PR and mark it ready

If the user passed a PR number as `$ARGUMENTS`, use it. Otherwise detect the PR for
the current branch:

```
gh pr view [number] --json number,title,url,headRefName,state,isDraft
```

If no PR is found or it is already merged/closed, tell the user and stop. If it's
still a draft, mark it ready (this is also what triggers CodeRabbit's auto-review):

```
gh pr ready {number}
```

If it's already ready, that's fine — continue.

## Step 2: Trigger review — CodeRabbit first, the fallback only on throttle

Marking the PR ready makes **CodeRabbit** auto-review. **Do not summon the
fallback reviewer by default** — it is only for when CodeRabbit is rate-limited.

Give CodeRabbit ~30–60s, then read its result on the head (see *CodeRabbit's result
on the head*). It is **throttled** when that result says *Rate limited*.

Don't judge throttling from CodeRabbit's comments. It edits one summary comment in
place and leaves older ones behind, so a *"rate limit"* or *"Review limit reached"*
comment can be about an earlier push. The reverse happens too: a fresh push can be
throttled while the latest comment still shows an earlier review.

**A throttled head is never reviewed later on its own.** Once CodeRabbit marks the
head *Rate limited*, it does not come back to it when its cooldown ends. It
reviews again only on a new push, or when someone asks it to, which this skill
never does (see below). So waiting out the cooldown achieves nothing. Fall back
now (below), and don't apply the long wait for the draft race.

**IMPORTANT — the "Draft detected" race is NOT a skip to act on.** If CodeRabbit
posts *"Review skipped — Draft detected"* (or *"Draft PR not reviewed"*) right after
you ran `gh pr ready`, that just means its webhook fired against the pre-ready state —
it hasn't processed the `ready` event yet. Do **not** fall back on this. CodeRabbit
reprocesses the `ready` event on its own and posts a real review.

**Budget tens of minutes for that, not ~60s.** Observed in practice: the draft-skip
comment at 18:01, the actual review at 18:51 — a **50-minute** gap, with no
"processing" comment in between to show it was alive. A 5- or 10-minute poll window
will expire while the review is still coming and make a perfectly healthy CodeRabbit
look stuck. Wait at least ~60 minutes before concluding anything, and prefer a
long-running watcher over a bounded poll loop.

While you wait, a later run on the head replaces the draft-time *Review skipped*
result: the latest one is what counts. Only treat it as inconclusive if the head
turns *Rate limited* (act on that at once, as above), or if, after that wait, the
latest result is still *Review skipped* for a non-draft reason (e.g. path filters,
`.coderabbit.yaml`). Never manually `@coderabbitai review` to un-stick it — that
burns its (throttled) quota; fall back instead.

- **Throttled / genuinely skipped, and `LAND_PR_FALLBACK_COMMENT` is set** → post it
  **exactly**, with no other prose, and note the time you posted it:
  ```
  gh pr comment {number} --body "$LAND_PR_FALLBACK_COMMENT"
  ```

  **Re-check CodeRabbit in the same breath as posting — not from an earlier poll.**
  Summoning the fallback spends real money (one observed run: **$7.29**) and cannot
  be recalled once the bot acknowledges, which takes seconds. A poll result goes
  stale the moment it is printed, so a loop that exits "no review yet" is evidence
  about the past, not the present.

  **Immediately** before the summon, re-read CodeRabbit's result on the head (see
  *CodeRabbit's result on the head*). Abort if it now says *Review completed* or
  *Review in progress*: CodeRabbit got to this head after all, so do **not** summon
  the fallback. This actually happened: a watcher's last poll missed the review by
  seconds, the summon went out **15 seconds after** CodeRabbit posted, and the money
  was spent for nothing. The check costs one API call; skipping it costs the user.

  Anchor that check on the head's result, not on counts of reviews or review
  comments on the PR. Those include every earlier push's review, and CodeRabbit's
  replies on review threads file review objects of their own (state `COMMENTED`).
  On any PR past its first round, the counts are non-zero whether or not this head
  was reviewed, so a guard built on them blocks every summon.
- **Throttled / genuinely skipped, and no fallback is configured** → tell the user
  CodeRabbit is throttled on this head and won't return to it by itself (only a new
  push, or the user asking, starts another review), and stop (see Configuration).
- **Reviewing (incl. after a draft-race re-check)** → let CodeRabbit run; do **not**
  also summon the fallback (avoid a double review).

## Step 3: Monitor the reviewer until it's done

Poll (~every 30s) until the active reviewer finishes — and give it room: a review
can take tens of minutes to land (see the 50-minute gap noted in Step 2), so a
bounded few-minute loop will expire long before a healthy reviewer is done. Prefer a
watcher that runs until it sees a result over a loop with a short deadline.

**CodeRabbit DOES re-review when you push** — do not assume the first review is the
last word. Observed: after pushing fixes for its first-pass findings, it ran a second
pass and found a further real bug (a correctness issue in one of the fixes itself).
Treat every push as starting a new review that has to finish before you merge. The
configured fallback may behave differently — it reviews a specific head SHA, so its
findings can silently predate your latest push; check the head it reports.

- **CodeRabbit done:** CodeRabbit has **completed a review of the current head
  commit** — not of an earlier one, and not merely started one. Done means its result
  on the head (see *CodeRabbit's result on the head*) says *Review completed*. Then
  read its findings from the review and inline comments whose `commit_id` is `$HEAD`.
  If the head turns *Rate limited* while you wait, it won't be reviewed: go back to
  Step 2's fallback instead of waiting on.

  **Do not accept any of these as "done"** — each one has been mistaken for it:
  - **The PR-level check** (`gh pr checks`). It aggregates across pushes, so it can
    show *"Review completed"* for a commit that is no longer the head.
  - **A `success` result that says "Review skipped".** Pushing while the PR is a
    draft attaches a *skipped* result to that commit, and its state is `success` (a
    check run's conclusion is too). A check on state alone reads it as a finished
    review of code nobody reviewed.
  - **A review object on the head commit.** CodeRabbit files a placeholder review
    (state `COMMENTED`, empty body) within seconds of starting, while the head's
    result still reads *"Review in progress"*. Its replies on review threads file
    review objects too. A review object on the right commit proves only that
    CodeRabbit said something there.
  - **No commit status at all.** Where CodeRabbit reports as a check run, the head
    never gets a status, so a watcher that reads only statuses waits forever and
    makes a finished review look stuck.

  Each of these fooled a watcher. The head's result (its status description, or its
  latest check run's summary) is the only signal that distinguishes skipped, in
  progress, rate limited and completed.
- **Fallback done:** match the bot by `LAND_PR_FALLBACK_LOGIN` (or the handle taken
  from the summon comment). The bot's review must be **of the current head**, as with
  CodeRabbit. A PR summoned on more than once still carries the earlier run's "done"
  comment, so a pattern match alone finishes the wait before the new review has
  begun. With `LAND_PR_FALLBACK_DONE` set, it is done once the bot's latest comment
  matches that pattern **and** names the head. Bots that edit one status comment in
  place put the commit they reviewed in it, usually as a short SHA:
  ```
  HEAD=$(gh pr view {number} --json headRefOid --jq .headRefOid)
  gh api repos/{owner}/{repo}/issues/{number}/comments \
    --jq '[.[]|select(.user.login|test("<login>";"i"))]|last|.body // ""' \
    | grep -iE '<done>' | grep -q "$(echo "$HEAD" | cut -c1-7)" && echo done
  ```
  If the bot's comment never names a commit, require one updated after your summon
  (its `updated_at`) instead. Without `LAND_PR_FALLBACK_DONE`, it is done once the bot
  has submitted a PR review on the head, or posted a comment after your summon:
  ```
  gh api repos/{owner}/{repo}/pulls/{number}/reviews \
    --jq "[.[]|select(.user.login|test(\"<login>\";\"i\"))|select(.commit_id==\"$HEAD\")]|length"
  ```

## Step 4: Address the review comments

Copied from the **pr-comments** skill.

**Check out the PR's branch first** if you aren't on it (a passed PR number often
means you're on `main`, so edits would land on the wrong branch):

```
git fetch origin <headRefName> && git checkout <headRefName> && git pull --ff-only
```

If uncommitted changes block the checkout, STOP and tell the user — do not stash or
discard.

Fetch the file-level review comments and the review threads (to know resolved vs not):

```
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate \
  --jq '.[] | {id, path, line, original_line, body, user: .user.login, in_reply_to_id}'

gh api graphql -f query='
{ repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {number}) {
      reviewThreads(first: 100) { nodes {
        id isResolved
        comments(first: 10) { nodes { databaseId path originalStartLine body author { login } } }
      } }
    } } }'
```

Then, for each **unresolved** thread (skip resolved threads and reply comments with
`in_reply_to_id`):

1. Read the file around the commented line for full context.
2. Decide what change is requested and make it with Edit.
3. **Reply on that specific thread** — say what you changed, or, if you're
   deliberately not changing it, why. Reply on the thread itself with the REST
   comment id:
   ```
   gh api repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies -f body="..."
   ```
4. Resolve the thread (only once you've addressed it or confirmed it's a non-issue):
   ```
   gh api graphql -f query='mutation { resolveReviewThread(input: {threadId: "<thread_node_id>"}) { thread { isResolved } } }'
   ```

**Reply to each comment individually — NOT with one summary comment at the top of the
PR.** Review bots *learn* from the per-comment replies (accept/reject
signal on each finding), so a single top-level summary teaches them nothing. One reply
per thread, on the thread.

Bot findings are **advisory** — apply the good ones, and if one is wrong or needs a
design decision, reply saying so and flag it to the user; do **not** resolve that
thread. After addressing everything, **commit and push** (run the repo's precheck if it
has one). CI re-runs on the push; the reviewer does not.

## Step 5: Merge

Copied from the **merge-pr** skill.

**Before arming auto-merge, confirm no review is in flight.** `--auto` fires the
moment *required* checks pass. If the reviewer's check is not a required one, a
review still running will be overtaken by the merge and its findings will land on an
already-merged PR. This nearly shipped a real bug: auto-merge was armed while
CodeRabbit's second pass was still running, and that pass found a correctness defect
in the fixes — it only survived because the merge happened not to fire first.

Check, and wait if anything is still going. Read CodeRabbit's result on the head (see
*CodeRabbit's result on the head*), and the review threads:

```
gh api graphql -f query='
{ repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {number}) {
      reviewThreads(first: 50) { nodes { isResolved } } } } }'
```

Arm it only once the head's CodeRabbit result reads **"Review completed"** (see Step 3
for why neither `gh pr checks` nor a review object on the head is enough) — or, when
CodeRabbit was rate limited on the head, once the fallback has finished reviewing it —
and you have addressed what it found. If you armed it and a review is still running,
disarm rather than race it:

```
gh pr merge {number} --disable-auto
```

Then squash-merge with auto-merge and delete the remote branch in one step:

```
gh pr merge {number} --squash --delete-branch --auto
```

`--auto` fires the merge as soon as required checks are green and branch protection is
satisfied (or immediately if already green). A bot verdict of `CHANGES_REQUESTED` /
`NEUTRAL` is advisory and does not block — merge on **CI green + mergeStateStatus
CLEAN**. If the merge itself fails (conflict, unmergeable, auto-merge disabled at the
repo), tell the user and stop.

Once it has actually merged (`gh pr view --json state` shows `MERGED`), switch back and
clean up:

```
git checkout main && git pull
git branch -d <headRefName>   # ignore errors if already deleted by --delete-branch
```

If auto-merge only **queued** the merge (state still `OPEN`), tell the user it's queued
and stop — don't fast-forward `main` while the PR is still pending.

## Step 6: Confirm

Tell the user the PR was reviewed (by whom), how many findings you addressed, that it
merged, and that they're back on a clean `main`.
