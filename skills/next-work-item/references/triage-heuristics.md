# Triage Heuristics

Use these rules when ranking the next work item.

## Source confidence

Rank candidate sources in this order:

1. Explicit source-of-truth statements in repo docs.
2. Explicit user instruction for a source.
3. Issue tracker remote plus working CLI (`gh` or `glab`).
4. Local roadmap/backlog/TODO docs with clear active tasks.
5. Everything else.

If two sources conflict, prefer the explicit source-of-truth statement unless the user overrides it.

## Actionability ranking

Prefer work that is:

- unblocked;
- concrete;
- scoped to one deliverable;
- obviously ready to plan or implement;
- dependency-unlocking without being a vague umbrella.

Deprioritize work that is:

- blocked;
- research-only when the user did not ask for research;
- an umbrella/coordinator issue with no concrete acceptance criteria;
- archival, wishlist, or speculative.

## Issue tracker hints

Signals that usually indicate blocked or non-leaf work:

- labels like `blocked`, `research`, `spike`, `epic`, `umbrella`
- body text containing `blocked by`, `depends on`, `part of`, `follow-up to`, `after`
- titles that frame coordination instead of delivery

Signals that usually indicate good next candidates:

- explicit priority labels;
- bug/feature/enhancement labels aligned with user intent;
- concrete acceptance criteria;
- verification steps;
- issue text that says it unblocks later work.

## Local document hints

Strong positive markers:

- headings like `Next`, `Current`, `Planned`, `Backlog`
- unchecked boxes
- prefixes like `P0`, `P1`, `High`, `Medium`
- wording like `next`, `do first`, `before`, `must`, `blocked by`

Ignore or deprioritize:

- checked boxes
- `done`, `complete`, `shipped`, `archived`
- brainstorming sections with no ownership or sequence

## Output discipline

Always end with a single recommendation unless the source is too ambiguous to justify one. When
ambiguity is high, present the top few candidates and explain why a forced single pick would be
speculative.
