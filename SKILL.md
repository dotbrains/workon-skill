---
name: workon
description: Pick up a Linear ticket end-to-end — worktree, implement, PR, then watch the PR on a 5-min loop addressing Codex comments, CI failures, and merge conflicts until merged. Use when the user types `/workon <TICKET-ID>` (e.g. `/workon ENG-66`).
argument-hint: "<TICKET-ID>"
---

# Workon: end-to-end ticket driver

Idempotent, state-branched skill. Same invocation drives three phases:

- **Setup** — no worktree yet → create worktree, read ticket, implement, PR, start loop.
- **Watch** — worktree + open PR → address Codex comments, fix CI, resolve conflicts, check convergence, check merge.
- **Teardown** — PR merged → remove worktree, stop loop.

**Arguments:** "$ARGUMENTS"

## 0. Parse argument

Extract `<TICKET-ID>` (e.g. `ENG-66`). Must match `[A-Z]+-\d+`. If missing or malformed, abort with a short error telling the user the expected form.

## 1. Load state

State file: `~/.claude/workon/<TICKET-ID>.json`. Shape:

```json
{
  "ticketId": "ENG-66",
  "worktreePath": "/absolute/path/to/worktree",
  "branchName": "feat/...",
  "baseBranch": "main",
  "repoSlug": "owner/repo",
  "prNumber": 123,
  "phase": "setup|watch|teardown",
  "convergenceCommentPosted": false,
  "lastAddressedCommentISO": null,
  "loopStarted": false
}
```

Create the `~/.claude/workon/` dir if it doesn't exist. If no state file, this is a fresh Setup run. If state exists, jump to the phase it declares and re-verify against GitHub — phase transitions are owned by the skill, not the state file (state is a cache, GitHub is source of truth).

## 2. Route

```text
if no state file OR phase == "setup":
    run Setup (§3)
elif phase == "watch":
    # Re-verify PR is still open; if merged, route to Teardown
    if pr.state == "MERGED": run Teardown (§5)
    else: run Watch (§4)
elif phase == "teardown":
    # Already done — no-op and hint at /loop-stop
    print "workon <ticket>: already torn down. Run /loop-stop to end the loop."
    exit
```

---

## 3. Setup phase

### 3.1 Load the ticket

Use your configured Linear integration to fetch the issue and its latest comments.

Synthesize scope using current status fields and recent comments, not only title/summary.

If the ticket is ambiguous, under-specified, or blocked on a decision, stop and tell the user. Do not guess, do not create the worktree yet.

### 3.2 Determine repo metadata

- `repoSlug`: `gh repo view --json nameWithOwner --jq .nameWithOwner`
- `baseBranch`: `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name` (do **not** hardcode `main` or `master`)
- Repo root: `git rev-parse --show-toplevel`

### 3.3 Create the worktree

- Branch name: use conventional prefixes (`feat` / `fix` / `chore`) and append the ticket id for auto-linking: `<type>/<slug>-<ticket-id-lower>`.
- Worktree path: `<repo-parent>/<repo-basename>-wt-<TICKET-ID>`.

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
PARENT=$(dirname "$REPO_ROOT")
REPO_NAME=$(basename "$REPO_ROOT")
WT="$PARENT/${REPO_NAME}-wt-${TICKET_ID}"
git -C "$REPO_ROOT" fetch origin "$BASE_BRANCH"
git -C "$REPO_ROOT" worktree add -b "$BRANCH" "$WT" "origin/$BASE_BRANCH"
```

Write the state file with `phase: "setup"` so crashes can resume cleanly.

### 3.4 Implement the ticket

Work inside the worktree (`cd "$WT"`). Run autonomously in this phase — no user check-in before opening the PR.

- Read local contributor docs (`CLAUDE.md`, `AGENTS.md`, etc.) and nearby code before starting.
- Reuse existing patterns before introducing new libraries/APIs.
- Work in small commits using conventional commit types.
- Run project quality gates locally before pushing.

If implementation requires a product decision outside ticket scope, stop, leave a WIP commit, notify the user, and exit.

### 3.5 Push and open PR

Use your normal PR workflow. Required behavior:

- Base branch = repo default branch from §3.2.
- PR title: `<type>: <short description> [<TICKET-ID>]`.
- Respect `.github/pull_request_template.md` when present.
- Keep description concise.
- Assign to the current user.

Record `prNumber` in the state file.

### 3.6 Kick off the watch loop

Set `phase: "watch"`, `loopStarted: true`, then invoke:

```text
/loop 5m /workon <TICKET-ID>
```

Exit setup. The loop drives §4.

---

## 4. Watch phase

Runs once per 5-minute tick. Fast no-op when nothing is actionable. Do §4.1–§4.5 in order each tick.

Context: `cd "$worktreePath"`. Refresh with `git fetch origin`.

### 4.1 Check merge state

```bash
gh pr view "$PR" --json state,mergedAt,mergeable,mergeStateStatus
```

- `state == "MERGED"` → route to Teardown (§5).
- `state == "CLOSED"` (not merged) → post a note on the Linear ticket, set `phase: "teardown"`, remove worktree, exit.
- Otherwise continue.

### 4.2 Fix merge conflicts

If `mergeable == "CONFLICTING"`:

```bash
git fetch origin
git merge "origin/$BASE_BRANCH"
# resolve conflicts in-place, then:
git add -A
git commit
git push origin HEAD
```

If conflict resolution requires product-level judgment, leave the worktree in conflict state, notify the ticket owner, and exit this tick.

### 4.3 Address Codex comments

Fetch comments newer than `lastAddressedCommentISO`:

```bash
# Issue comments (general PR comments)
gh api "repos/$REPO/issues/$PR/comments" --paginate \
  --jq '[.[] | select(.user.login | test("codex|chatgpt"; "i"))]'

# Review comments (line-level)
gh api "repos/$REPO/pulls/$PR/comments" --paginate \
  --jq '[.[] | select(.user.login | test("codex|chatgpt"; "i"))]'
```

For each new Codex comment:

1. If valid, implement the fix.
2. If incorrect or out of scope, reply with rationale.
3. Commit fixes.
4. Resolve review threads explicitly after pushing.
5. Update `lastAddressedCommentISO` to newest processed comment.

Push once at the end of §4.3.

### 4.4 Fix CI failures

```bash
gh pr checks "$PR"
```

- Ignore non-blocking informational checks.
- If required checks are red, fetch failed logs (`gh run view <run-id> --log-failed`), diagnose, fix, commit, push.
- If failure appears unrelated to this PR, rerun failed jobs once (`gh run rerun <run-id> --failed`) before coding a fix.

### 4.5 Convergence check

Read last commit timestamp and last Codex comment timestamp:

```bash
LAST_COMMIT=$(gh api "repos/$REPO/pulls/$PR/commits" --jq '[.[]][-1].commit.committer.date')
LAST_CODEX=$(gh api "repos/$REPO/issues/$PR/comments" --paginate \
  --jq '[.[] | select(.user.login | test("codex|chatgpt"; "i")) | .created_at] | last')
```

Converged when both:

1. `now - LAST_COMMIT >= 30 minutes`, and
2. `LAST_CODEX` is null or `LAST_CODEX < LAST_COMMIT`.

If converged and `convergenceCommentPosted == false`, post a concise Linear update with PR URL and set `convergenceCommentPosted: true`.

Continue watching after convergence — humans may still comment and CI/conflict state may change.

---

## 5. Teardown phase

### 5.1 Final Linear comment (optional)

If PR merged and no convergence comment was posted, leave a short merge confirmation with PR URL. Otherwise skip.

### 5.2 Remove the worktree

```bash
REPO_ROOT_MAIN=$(git -C "$worktreePath" rev-parse --git-common-dir | xargs dirname)
cd "$REPO_ROOT_MAIN"
git worktree remove --force "$worktreePath"
git branch -D "$branchName" 2>/dev/null || true
```

### 5.3 Stop the loop

Set `phase: "teardown"` in state so subsequent ticks no-op.

Then try, in order:

1. `/loop-stop` (or equivalent).
2. Scheduler API delete for the matching `/workon <TICKET-ID>` entry.
3. If neither is available, print: **"PR merged, worktree removed. Run /loop-stop to end the watch loop."**

### 5.4 Exit

Print a two-line summary: PR URL + "merged and cleaned up."

---

## Cross-cutting rules

- **One push per tick.** Batch fixes and push once; each push resets the 30-minute convergence clock.
- **Never force-push** unless the user explicitly asks for history rewrite.
- **No speculative ticket creation.** Fix in-scope issues or leave concise open questions for the user.
- **Avoid internal jargon** in ticket comments; write clear external-facing language.
- **State file is a cache.** GitHub and filesystem are sources of truth.
- **Idempotency first.** Setup/watch/teardown must be safe to re-enter.
