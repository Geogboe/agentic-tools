---
name: next-work-item
description: >
  Finds the next thing to work on in a repository by detecting the most credible planning source,
  triaging it, and recommending one actionable next item. Use this skill when the user says things
  like "let's work on the next thing", "next feature", "next fix", "what should we work on next",
  "pick the next issue", or asks you to triage a backlog, roadmap, or issue tracker to choose the
  next task. It supports GitHub issues, GitLab issues when glab is available, and repo-local
  roadmap/backlog/TODO docs. If the user names another source, use it when possible or ask for the
  artifact/path needed to inspect it.
---

# Next Work Item

Pick the next actionable work item, explain why, and hand off directly into planning. Do not jump
into implementation yet.

## What this skill does

1. Detect the current repo and likely planning sources.
2. Recommend the best source based on repo evidence.
3. Ask the user to confirm or override the source.
4. Triage that source for the requested intent.
5. Return one recommended item with rationale and planning handoff.

Read `references/triage-heuristics.md` before ranking candidates.

## Intent mapping

Interpret the user's phrasing before triaging:

- `next fix`, `bug`, `broken`, `repair` => prioritize fixes.
- `next feature`, `enhancement`, `build` => prioritize features.
- `research`, `spike`, `investigate` => allow research items.
- Generic `next thing`, `what should we work on next` => pick the best actionable item overall.

If the user explicitly names a source, honor it instead of auto-detecting.

## Step 1: Establish repo context

Start by proving what environment you are in:

```bash
git rev-parse --show-toplevel
git remote -v
```

If this is not a git repo, say so and ask the user what source to inspect.

## Step 2: Discover candidate planning sources

Inspect local instructions and likely planning docs first:

```bash
rg -n "single source of truth|roadmap|backlog|todo|next|issue tracker|GitHub issues|GitLab issues" \
  AGENTS.md CLAUDE.md README* docs/ ROADMAP* BACKLOG* TODO* 2>/dev/null
```

Treat these as credible sources:

- Explicit source-of-truth statements in `AGENTS.md`, `CLAUDE.md`, `README*`, or docs.
- GitHub remotes when `gh` is available.
- GitLab remotes when `glab` is available.
- Local planning artifacts such as `ROADMAP*`, `BACKLOG*`, `TODO*`, and backlog sections in docs.

Treat these as weaker signals:

- A GitHub or GitLab remote with no explicit planning statement.
- Architectural docs that mention long-term ideas but do not act like a backlog.

If there is an explicit source-of-truth statement, prefer it over inferred signals.

## Step 3: Recommend a source, then confirm

Default policy is detect first, then ask.

Use this pattern:

```text
I found these planning signals:
- GitHub remote: owner/repo
- AGENTS.md says GitHub issues are the single source of truth

Recommended source: GitHub issues.
Unless you want a different source, I'll triage that now.
```

If the user already asked for a specific source, skip this confirmation and use that source.

If no credible source is found, ask the user what source to use. If they mention another system
such as Jira, Linear, a roadmap doc, or a notes file, ask for the specific artifact, path, or
access method you can inspect.

## Step 4: Triage the chosen source

### GitHub issues

Use `gh` against the repo's open issues. Start broad, then inspect promising candidates in detail.

Typical flow:

```bash
gh issue list --limit 100 --state open
gh issue view <number>
```

Ranking rules:

- Exclude blocked items by default.
- Exclude research items unless the user asked for research.
- Rank by explicit priority labels first.
- Then rank by actionability: concrete leaf tasks beat umbrella issues.
- Then apply the user's intent:
  - `fix` prefers bugs.
  - `feature` prefers feature/enhancement work.
  - generic requests can choose either.

When issue bodies mention `blocked by`, `depends on`, `part of`, or `blocks`, use that dependency
text when deciding whether the issue is truly actionable now.

### GitLab issues

Only use GitLab issue triage when the repo is on GitLab and `glab` is installed.

Typical flow:

```bash
glab issue list
glab issue view <iid>
```

Apply the same ranking and blocker rules as GitHub.

If the repo is on GitLab but `glab` is not installed, say that clearly and offer the best
available fallback source instead of pretending to support it.

### Local roadmap / backlog docs

Scan only credible planning files, not every markdown file in the repo.

Prefer:

- `ROADMAP*`
- `BACKLOG*`
- `TODO*`
- backlog or roadmap sections in `AGENTS.md`, `CLAUDE.md`, `README*`, or `docs/**`

Ranking rules:

- Prefer explicit markers such as `next`, `todo`, `priority`, `blocker`, and unchecked task boxes.
- Ignore completed, archived, or struck-through items.
- Use document order as a fallback only when priority markers are absent.
- If dependencies are mentioned in prose, do not recommend blocked items.

If the docs are too ambiguous to support a defensible single pick, return a short ranked shortlist
and ask the user to choose instead of guessing.

## Step 5: Return one recommendation

Your output should be concise and decision-oriented.

Include:

- `Recommended next item`
- `Source used`
- `Why this is next`
- `Dependencies or blockers`
- `Why other obvious candidates were not chosen`
- `Planning handoff`

Example shape:

```text
Recommended next item: #66 Add async sandbox creation model to API
Source used: GitHub issues for owner/repo

Why this is next:
- unblocked
- priority:high
- concrete implementation task
- unlocks follow-on CLI migration work

Dependencies or blockers:
- no blocking dependency called out
- it unblocks #67

Not chosen:
- #34 is blocked
- #85 is actionable but lower leverage than the architecture-critical API issue

Planning handoff:
I'll plan #66 next unless you want to switch to a bug-first pass.
```

## Guardrails

- Do not fabricate certainty when the source is ambiguous.
- Do not recommend blocked work unless the user explicitly wants blocked or dependency work.
- Do not bury the recommendation inside a long backlog summary.
- Do not start implementing. This skill stops at selection plus planning handoff.
- If the user asks for another source, adapt instead of forcing GitHub/GitLab/docs.
