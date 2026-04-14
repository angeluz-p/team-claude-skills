# pr-reviewer (skill) — Design Document

A Claude Code **skill** that reviews GitHub pull requests. Produces a structured report (scorecard, findings by priority, verdict) and optionally cross-validates with OpenAI's Codex. Safe to run anywhere — never edits code, never posts without permission.

Companion to `pr-reviewer-design.md` (agent variant). This doc covers the **skill** variant, which is the default for everyday reviews.

---

## 1. Why a skill, not a subagent

The original `pr-reviewer` was built as a subagent (`~/.claude/agents/pr-reviewer.md`) to get tool-allowlist enforcement — the agent frontmatter excludes `Edit` and `Write`, making it structurally impossible to modify code.

That tradeoff turned out to be expensive in practice. Across five consecutive runs, the subagent relay compressed the full report into a 3-5 bullet summary before showing it to the user, and `AskUserQuestion` prompts were flattened into inline text instead of firing as interactive pickers. The user had to prompt "show me the full report" or "did you ask about Codex?" on almost every run.

**Root cause:** subagents return their final message to the parent Claude as a tool result. The parent decides how to present that result. Even when the subagent produced the full report internally, the parent summarized it on the way out. No amount of prompt tightening inside the agent file fixed this — the problem was structural, not instructional.

**The skill variant solves this by running in the user's main conversation.** No relay, no parent summarizing, no compression. What the skill renders is what the user sees. `AskUserQuestion` fires as real interactive prompts.

### What we traded

| Dimension | Subagent (agent) | Skill |
|---|---|---|
| Report rendering | Compressed by parent relay | Rendered directly in chat |
| `AskUserQuestion` | Often flattened to inline text | Fires as interactive picker |
| Tool-allowlist enforcement | Structural — `Edit`/`Write` not in frontmatter | Structural — `allowed-tools` frontmatter excludes `Edit`/`Write`/`NotebookEdit` |
| Context cost | Isolated (~35k tokens don't hit main conversation) | Lands in main conversation |
| Reliability | Low (5/5 runs failed a mandatory step) | High — mandatory question enforcement proven across multiple runs |

The skill is the right default for normal PRs. The agent stays as a fallback for **huge PRs** (where context cost matters) or **high-stakes reviews** (where structural tool-level safety is worth the UX penalty).

---

## 2. The problem

Manual PR reviews have two failure modes:

- **Rushed review** — skim the diff, miss things, approve. Bugs slip through.
- **Thorough review** — take 30-60 minutes per PR, run out of time to do it on every PR.

A good reviewer does the boring systematic pass (unused imports, missing null checks, scope drift, naming, test coverage) AND the judgment pass (is this the right architecture, does this belong here, is the abstraction earning its keep). Doing both manually, every time, is unrealistic.

**Goal:** automate the systematic pass so human time goes to judgment, not grep.

---

## 3. Core design principles

Every decision ladders up to one of these four:

1. **Report-only, never auto-edit.** A reviewer that edits code is a co-author. Different job.
2. **User sovereignty.** Ask before anything public, destructive, or expensive.
3. **Safety with branch state.** Never lose uncommitted work. Never surprise-switch branches.
4. **Match effort to stakes.** Low-stakes PRs get a fast review. High-stakes PRs get a second opinion.

---

## 4. Key design decisions (and why)

### 4.1 Why report-only, not auto-fix

The skill reviews *someone else's* PR. Pushing commits to someone else's branch without asking is a trust violation.

**Enforcement is structural.** The skill's frontmatter declares `allowed-tools: Bash, Read, Grep, Glob, AskUserQuestion` — `Edit`, `Write`, and `NotebookEdit` are not in the list, so the skill physically cannot call them. A reviewer that can't edit can't misbehave. This matches the agent variant's safety model; we don't lose anything structural by running as a skill.

The `SKILL.md` instructions also state "NEVER use Edit, Write, or NotebookEdit" as an absolute rule, but that's belt-and-suspenders — the frontmatter does the real work.

### 4.2 Why Claude-first, Codex opt-in (not both always, not Codex always)

Codex is a separate AI model from OpenAI. Running it gives two independent reviews instead of one, catching ~15-20% more findings than any single model.

But Codex has real costs:
- Needs the PR branch **checked out locally**
- Takes extra time (~60 seconds per review)
- Costs a separate OpenAI API call

**Three modes considered:**

| Option | Behavior | Verdict |
|---|---|---|
| Always run Codex | Max coverage, max cost | Too aggressive — most PRs don't need it |
| Never run Codex | Claude only, simple | Too limited — high-stakes PRs benefit from cross-validation |
| **Opt-in after first review** | Claude runs → shows findings → asks about Codex | **Chosen.** Informed decision, not preemptive commitment |

**Why "after" not "before":** asking "do you want a second opinion?" before any review has happened forces a blind decision. Asking after shows Claude's findings first, so the user can judge whether Codex is worth the cost for this specific PR.

**When to say yes to Codex:** auth/authz changes, database migrations, payment/billing logic, public API contracts, infrastructure changes, unfamiliar codebase.

**When to say no:** docs, typos, log tweaks, small UI polish, well-contained refactors with tests, re-reviews where context is already known.

### 4.3 Why remote mode is the default

Remote mode reads the diff via `gh pr diff <PR>` — no checkout, no branch switch. The skill sees the same diff either way (byte-for-byte identical) and produces the same quality review.

**Why this matters:**
- Safe to run anywhere, any time, even mid-work on another branch
- No stash/restore gymnastics for 80% of reviews
- Zero blast radius if something goes wrong mid-skill

**Checkout is only triggered** when the user explicitly opts into Codex (Step 4). Outside of that path, the skill never touches the local branch.

### 4.4 Why `/qa-only` is gated, not default

`/qa-only` is a gstack skill that tests a running web app via headless Chromium browser automation. It's a **different tool** than code review — it verifies deployed behavior, not code correctness.

**Why not default:**
1. Most PRs aren't web-UI changes (infra, backend, docs, config). Browser tests on them produce noise.
2. Requires a **live staging/preview URL.** Most PRs don't have one.
3. Requires the gstack `browse` binary installed and built.
4. Slow (minutes per run) — overkill for a typo fix.
5. Pure code review answers "is the code correct?" — doesn't need a browser.

**Why not removed entirely:** live testing catches a different class of bug — runtime errors from missing env vars, broken migrations that look fine in the diff, visual regressions, API contract breaks.

So `/qa-only` stays, but is tightly gated: only triggers if a staging URL is auto-detected in the PR body, OR the user explicitly says `"with qa"`. Otherwise skipped silently.

### 4.5 Why the 15-point scorecard uses percentages (not raw counts)

A fixed rubric across all PRs gives consistent vocabulary. But a 1-file docs PR and a 2000-line backend refactor shouldn't be scored against the same 15 checks.

**Solution:** any check can be marked **N/A** when it doesn't apply. Denominator is `applicable = 15 − N/A count`. Verdict thresholds are percentage-based:

- **100%** → APPROVE
- **80-99%** → APPROVE with minor comments
- **55-79%** → NEEDS DISCUSSION
- **<55%** → REQUEST CHANGES

**Overrides (bypass percentage):**
- Any security/auth check FAIL → automatic REQUEST CHANGES
- Any P0 finding → automatic REQUEST CHANGES (severity wins over breadth)

Sanity floor: tiny PRs (<5 applicable checks) default to NEEDS DISCUSSION if there's any finding — the percentage is unreliable on small samples.

### 4.6 Why re-review memory tracks only the user's prior reviews

On re-reviews, the skill fetches prior reviews and compares them against the current diff:
- **Resolved** — skip, don't re-flag
- **Still unresolved** — include with `[previously raised]` tag
- **Status unclear** — flag for user to clarify

**Why filter to only the user's reviews (not everyone's):** the skill is *your* memory across re-reviews, not a team aggregator. Incorporating other reviewers' comments would amplify Copilot's false positives, re-state things already discussed by teammates, and confuse the "what changed since *my* last review" signal.

User handle is auto-detected via `gh api user --jq .login` — no hardcoding, so the skill works for anyone who installs it.

### 4.7 Why safety gates around branch manipulation

When the user opts into Codex (and checkout is required), safety gates run in this order:

1. **Preflight** — check `git`, `gh`, `codex` are installed
2. **Remember original branch** — so we can restore it
3. **Dirty-tree check** — if uncommitted work exists, ask: stash / commit / cancel
4. **Conflict pre-flight** — compare user's dirty files against PR's changed files. If they overlap, **refuse to stash** and offer to cancel Codex instead. Prevents predictable merge conflicts on stash pop.
5. **Stash with `--include-untracked`** — captures new files too
6. **Checkout PR, run Codex, merge findings** — state-changing commands run as separate bash calls, never chained, so each transition is individually approvable
7. **Auto-cleanup:** switch back to original branch → stash pop → tell user explicitly "restored"
8. **Stash-pop conflict handler** — if pop fails, stop, keep stash safe, tell user where it is

**Principle:** the skill should be more conservative than the user, not less. If anything looks risky, bail out with the Claude-only report.

### 4.8 Why the full report renders twice in the Codex path

After Codex runs (success or failure), the stash/checkout/cleanup sequence produces a lot of tool output between the original Step 6 report and the pending post-to-GitHub question. The user would have to scroll back to see the scorecard before deciding how to post.

**Solution:** if Codex was attempted, re-render the full Step 6 report as a fresh message before firing the post-to-GitHub `AskUserQuestion`. The re-rendered report has the Reviewer line updated:
- Codex succeeded → "Claude <model>, cross-validated by Codex CLI"
- Codex failed → "Codex attempted but <reason>, findings from Claude only"

Skipped when Codex was never attempted — in that case the original Step 6 render is still the most recent visible message.

### 4.9 Why bash commands run separately, not chained

Every state-changing command (stash, checkout, post) runs as its own `Bash` call — never chained with `&&` or `;`, never redirected with `>` or `>>` to a file. Three reasons.

**User-level approval granularity.** Each transition is independently approvable. The user can approve a stash but decline a checkout if they change their mind mid-flow. Chained commands collapse multiple state changes into one approval, which violates the "user sovereignty" principle.

**Settings allowlist compatibility.** The user's `~/.claude/settings.json` allows read-only commands like `Bash(gh pr diff:*)` and `Bash(git status:*)` without prompting. Claude Code's permission matcher evaluates the whole bash string against each allowlist entry — so a chain like `gh pr diff 280 > /tmp/diff && wc -l /tmp/diff && gh pr diff 280 --name-only` matches no single entry and forces a prompt, even though every piece is read-only. Running each as a separate call lets the allowlist do its job.

**No file redirects.** `>` and `>>` are write operations, even to `/tmp`. On a read-only allowlist they're denied. Read the diff via stdout, parse in-memory, don't write scratch files.

---

## 5. Skill flow

```
User: "/pr-reviewer 280"
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ Step 0: Preflight                                    │
│ - Check git, gh, codex installed                     │
│ - Verify gh is authenticated                         │
│ - Confirm we're in a git repo                        │
└─────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ Step 1: Acquire PR (remote mode — no checkout)       │
│ - gh pr checkout NOT called. Branch untouched.       │
└─────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ Step 2: Read PR metadata                             │
│ - title, body, labels, author, isDraft               │
│ - reviews, comments (re-review memory, your handle)  │
│ - diff via gh pr diff                                │
│ - commit messages                                    │
│ - Auto-detect staging URL in PR body                 │
│ - Stop if draft, confirm before proceeding           │
└─────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ Step 3: Claude reviews the diff                      │
│ - 15-point scorecard (with N/A where applicable)     │
│ - Findings by priority (P0/P1/P2/P3)                 │
│ - Scope drift check                                  │
│ - What's Good section                                │
└─────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ Step 6 (early render): Full report to chat           │
│ - Scorecard table, all sections, all P0/P1/P2/P3     │
│ - Rendered directly in user's conversation           │
└─────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ Step 4: MANDATORY AskUserQuestion about Codex        │
│ - If yes → safety gates → checkout → Codex →        │
│          auto-cleanup → re-render updated report     │
│ - If no → report above is final                      │
└─────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ Step 5: Live testing (tightly gated, usually skipped)│
└─────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────┐
│ Step 7: THREE MANDATORY AskUserQuestion calls        │
│                                                      │
│ Q0: Edit/remove findings? Yes → reword/drop items    │
│                                                      │
│ Q1: Post inline? Yes → each finding pinned to its    │
│     exact file:line on the diff (P0/P1/P2/P3 label) │
│                                                      │
│ Q2: Post summary? Approve / Request Changes /        │
│     Comment / No → scorecard + verdict as body       │
│     (recommended option matches verdict)             │
│                                                      │
│ - Never auto-posts. All three questions always fire. │
│ - Uses --body-file, never "#<number>" in bodies      │
└─────────────────────────────────────────────────────┘
```

---

## 6. Permission points

| When | Question | Why |
|---|---|---|
| Step 2 (conditional) | Draft PR — review anyway? | Reviewing drafts creates noisy findings |
| Step 4 (default) | Want Codex second opinion? | Informed opt-in for extra cost |
| Step 4 (conditional) | Uncommitted changes — stash / commit / cancel? | Safety before branch switch |
| Step 4 (conditional) | Files overlap with PR — commit / cancel? | Safety against stash-pop conflict |
| Step 5 (conditional) | Preview URL found — run `/qa-only`? | Opt-in for live testing |
| Step 7 Q0 (always) | Edit or remove findings before posting? | User controls what gets posted |
| Step 7 Q1 (always) | Post inline? | Opt-in for line-level comments on diff |
| Step 7 Q2 (always) | Post summary — Approve / Request Changes / Comment / No? | Never auto-post; verdict pre-highlights recommended option |

**Automatic (no prompt):**
- Step 1 remote-mode acquisition (never checks out)
- Branch restoration after Codex
- Stash pop after Codex
- Codex invocation once user has opted in

---

## 7. Safety guarantees

**What the skill will NEVER do:**
- Edit code (structurally impossible — `allowed-tools` frontmatter excludes `Edit`, `Write`, `NotebookEdit`)
- Push commits
- Auto-post to GitHub
- Switch your branch without explicit consent
- Drop or clear stashes
- Run `/qa` (the fix-the-bugs variant — uses `/qa-only` only)
- Write `#<number>` in PR comments (creates false backlinks)
- Chain state-changing commands (each runs as its own bash call)

**What the skill will DO automatically:**
- Read PR data, diff, commits, prior reviews
- Run Claude's review (always)
- Render the full report to chat
- Run Codex's review (only after user says yes)
- Auto-stash before checkout (only after user says yes)
- Auto-restore branch + stash after Codex
- Re-render report after Codex with updated Reviewer line
- Skip Codex if it fails mid-run (degrades gracefully to Claude-only)

**Failure modes and handling:**
- Codex fails / quota exhausted → keep Claude-only report, note failure in Reviewer line
- Browser stack missing → skip `/qa-only`, continue
- Stash pop conflict → stop, tell user work is safe in `git stash list`
- Dirty tree + file overlap with PR → refuse to stash, offer Claude-only
- Temp file empty before posting → stop, tell user to retry (don't post blank review)
- Approve body dropped by GitHub → auto-post body as follow-up `gh pr comment`

---

## 8. Tradeoffs acknowledged

**Pro:** Full report renders directly, mandatory questions fire natively, ~280 lines of plain text you can edit in 30 seconds.
**Con:** Context cost lands in main conversation (~30-40k tokens per review).

**Pro:** Safe, conservative, won't lose your work.
**Con:** More prompts than "just run it and trust me" tools.

**Pro:** Uses best-in-class models (Claude + optional Codex).
**Con:** Each review costs real money ($0.30-$2 depending on PR size, plus Codex's OpenAI bill if invoked).

**Pro:** Structural report-only enforcement via `allowed-tools` frontmatter — Edit/Write/NotebookEdit are not even loaded, so misbehavior is impossible, not just discouraged.
**Con:** If you later want the skill to edit something (e.g., auto-apply a suggestion), you'd have to broaden the allowlist and re-audit the safety story.

---

## 8.1 Posting flow

Step 7 uses a two-question flow instead of a single mode picker:

**Question 1 — Post inline?** Yes / No
- Yes → posts each finding as a line-level comment on its exact file:line, priority label included (P0/P1/P2/P3 visible on the diff). Uses `gh api repos/{owner}/{repo}/pulls/{number}/reviews` with a `comments[]` array.
- No → skip inline, go to Question 2.

**Question 2 — Post summary?** Approve / Request Changes / Comment / No
- Posts scorecard, verdict, what's good, scope check as a review body — no findings list (those are already inline).
- Recommended option is pre-highlighted based on verdict.
- No → done, keep everything in chat only.

This means the typical flow for an APPROVE review is: inline findings pinned to their lines + summary posted as Approve. Author sees the approval signal and can address the inline notes before merging or just merge.

**Why two questions:** inline and summary serve different purposes. Inline gives the author line-level context; the summary gives the overall verdict signal. Separating them lets you post inline without approving yet, or approve without inline if the PR has no findings worth pinning.

**Inline error handling:** if the GitHub API rejects a line (HTTP 422 — not in diff), that finding is removed from the inline payload and appended to the summary body under "Findings not mappable to diff lines." Full fallback to regular `--comment` if the whole inline call fails.

**Approve body-drop fix:** GitHub occasionally drops the review body when using `--approve --body-file` (confirmed bug, happened twice in practice). The skill verifies after posting via `gh pr view --json reviews`. If the body is missing, it auto-posts the report as a `gh pr comment` and tells the user.

---

## 9. Why skill, not subagent (final answer)

Interactive workflows that render structured output and ask multiple questions belong in skills. Every similar tool in the Claude Code ecosystem (`/review`, `/codex`, `/qa`, `/qa-only`) is a skill for the same reason — they're conversations with the user, not delegated tasks.

Subagents are the right container for autonomous, context-isolated jobs that produce a single result. PR review is not that. It has 4-6 decision points (draft check, Codex opt-in, stash decision, conflict response, QA opt-in, post mode) that require user input at runtime. Shoehorning that into a subagent produced the relay-compression and flattened-question bugs we saw across five consecutive runs.

**The agent variant (`~/.claude/agents/pr-reviewer.md`) is retained as legacy only.** The skill is the real pr-reviewer. If you see the agent file referenced in older documentation, prefer the skill.

Safety via tool frontmatter: the skill's `allowed-tools` excludes `Edit`, `Write`, and `NotebookEdit`. A reviewer that can't edit can't misbehave. This is the structural enforcement we originally went to the agent for — it works equally well on a skill.

---

## 10. Comparison

**vs GitHub Copilot code review:** closed-source, can't tune or inspect. This skill is 280 lines of plain text — fully auditable, fully modifiable.

**vs CodeRabbit / Graphite:** SaaS products with their own billing, privacy model, integration overhead. This skill runs locally via Claude Code you already pay for.

**vs Codex alone:** Codex is a strong reviewer but operates on current branch state only. This skill adds: safety gates, re-review memory, structured scorecard, remote-mode support, opt-in to Codex rather than forcing it.

**vs manual review:** not a replacement — a **supplement**. Skill runs the systematic pass; human does the judgment pass.

---

## 11. Cost expectations

Per review, rough estimates:

| PR Size | Claude tokens | Claude cost | Codex cost (if invoked) |
|---|---|---|---|
| Small (<300 lines) | 15-25k | $0.20-0.40 | ~$0.30-0.60 |
| Medium (300-1500) | 30-50k | $0.50-0.90 | ~$0.80-1.50 |
| Large (1500-5000) | 60-120k | $1-2 | ~$1.50-3 |
| Huge (5000+) | 150k+ | $2.50+ | $3+ |

For a team reviewing 20-50 PRs per week, roughly $40-150/month on Claude side, plus similar on OpenAI if Codex runs on all reviews. Most reviews don't need Codex, so real-world usage is lower.

---

## 12. Rollout plan

**Phase 1 (current):** Solo use + one pilot user (Eugene, already familiar with Codex CLI).
**Phase 2:** After pilot feedback, extend to 2-3 more technical team members.
**Phase 3:** Wider team, with SETUP.md as the onboarding doc.
**Phase 4 (if demand):** Consider a "lite" version for non-technical reviewers that produces a plain-English summary instead of the technical scorecard.

Feedback loop: after each user's first 3 reviews, ask what worked, what confused them, what they'd change. Iterate on the skill file.

---

## 13. Files

- `~/.claude/skills/pr-reviewer/SKILL.md` — the skill itself (primary)
- `.claude/skills/pr-reviewer/SKILL.md` — project-level copy (teammates get it on pull)
- `.claude/skills/pr-reviewer/DESIGN.md` — this file, co-located with the skill
- `~/.claude/agents/pr-reviewer.md` — the agent variant (deprecated, legacy only)
- `~/.claude/agents/pr-reviewer-SETUP.md` — install and usage guide
- `docs/pr-reviewer-skill-design.md` — canonical design doc (source of truth)

---

## 14. Invocation reference

```bash
# Skill (default):
/pr-reviewer 280

# Skill with shortcuts:
/pr-reviewer 280 with codex    # Run Codex, no question
/pr-reviewer 280 skip codex    # Claude-only, no question
/pr-reviewer 280 with qa       # Also run /qa-only on staging

# Agent (fallback for huge PRs or high-stakes reviews):
use pr-reviewer for PR 280
```

---

*Last updated: 2026-04-14.*
