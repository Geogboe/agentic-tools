---
name: ship-it
description: >
  Finalizes completed work by running the full ship-it checklist: test suite with coverage,
  code review, PII scan, clean conventional commits, push, PR creation, release pipeline
  monitoring, artifact installation, validation, and issue closure. Use this skill whenever
  the user says "ship it", "finalize this", "we're done let's push", "wrap this up",
  "let's land this", "validate and push", "ok finish up", "let's get this released",
  or otherwise signals that implementation is complete and they want to go through the
  full delivery pipeline. Also trigger when you see phrases like "ok move forward",
  "let's close this out", "merge and release", or "are we done?". Works on any stack —
  Go, Python, Node, Rust, or anything else.
---

# Ship It

You are finalizing completed work and shepherding it through the full delivery pipeline.
This is the last mile — the work is done, and now you need to validate, package, and
deliver it with confidence.

The philosophy: gather all the information you need first, ask the user any clarification
questions in one batch, then execute the entire pipeline without stopping. The user should
be able to kick this off and come back to a shipped release.

---

## Phase 0 — Gather Context

Before doing anything else, silently collect all of the following. Do not ask the user
yet — just observe.

1. **Git state**
   - Current branch, uncommitted changes, commits ahead of remote
   - Recent commit messages (are they already conventional?)
   - Whether this branch tracks a remote

2. **Project type**
   - Language/stack (check for `go.mod`, `package.json`, `Cargo.toml`, `pyproject.toml`,
     `requirements.txt`, `Gemfile`, `*.csproj`, etc.)
   - Test runner (detect from config files, scripts, Taskfile, Makefile, package.json scripts)
   - Coverage tooling (look for coverage configs, CI coverage steps, nyc, coverage.py,
     go test -cover, cargo-tarpaulin, etc.)

3. **GitHub/GitLab context**
   - Is this a GitHub repo? (`gh repo view` or check `.git/config` remote)
   - Does it have a release pipeline? (check for `release-please-config.json`,
     `.goreleaser.yml`, `.github/workflows/*release*`, semantic-release config)
   - Does it have install scripts? (`scripts/install.sh`, `scripts/install.ps1`,
     or a Taskfile `install` target)

4. **Issue context**
   - Scan the conversation history for issue references (`#123`, `fixes #`, `closes #`,
     issue URLs) that belong to THIS repository
   - Check the branch name for issue references (e.g., `feat/42-add-widget`)
   - Only consider issues for the current repo — ignore cross-repo references

5. **PR context**
   - Is there already an open PR for this branch?
   - What's the default branch? (usually `main` or `master`)

---

## Phase 1 — Ask Clarifications (One Batch)

Based on what you gathered, present a brief summary table of what you found, then ask
all clarification questions at once. Typical questions (skip any you can already answer):

- **PR approval**: "Should I approve the PR myself, or would you like to approve it?"
  Default to asking the user to approve.
- **Issue closure**: "I see this work relates to issue #X — should I close it when
  we're done?" Only ask if you found an issue reference. If the issue clearly belongs to
  the current project and the work clearly addresses it, you can state your intent to
  close it and let the user correct you rather than asking.
- **Coverage threshold**: "I'll check test coverage — is there a minimum % you care
  about, or should I just report what it is?" Only ask if there's no existing coverage
  config with a threshold.
- **Commit cleanup**: If the commit history has issues (missing prefixes, vague messages,
  WIP commits), summarize what you found and ask whether the user wants them cleaned up.
  Some people prefer to keep their history as-is — don't assume rewriting is welcome.

Wait for the user to respond before proceeding. Once you have answers, execute the
remaining phases without interruption.

---

## Phase 2 — Run Tests & Check Coverage

Run the project's test suite and capture coverage metrics. Adapt to the stack:

| Stack | Test command | Coverage |
|-------|-------------|----------|
| Go | `go test -coverprofile=coverage.out ./...` | `go tool cover -func=coverage.out` |
| Node | `npm test` or `npx jest --coverage` | Check stdout or `coverage/` dir |
| Python | `pytest --cov` or `python -m pytest --cov` | Check stdout |
| Rust | `cargo test` | `cargo tarpaulin` if available |
| Other | Check Taskfile, Makefile, or package scripts | Report what's available |

If a Taskfile exists with a `test` task, prefer `task test` as it may already include
coverage flags.

**Report findings**: show pass/fail count and coverage percentage. If coverage dropped
significantly compared to what's in CI config, flag it. If tests fail, stop and tell the
user — don't proceed with broken tests.

---

## Phase 3 — Code Review

**Invoke the `code-review:code-review` skill now.**

This gives the work a thorough review for bugs, security issues, code quality, and
adherence to project conventions. If the review surfaces blocking issues (bugs, security
vulnerabilities), stop and present them to the user before continuing.

Non-blocking suggestions (style, minor improvements) should be noted but should not
block the pipeline.

---

## Phase 4 — PII Scan

**Invoke the `repo-pii-scanner` skill now.**

This scans both current files and git history for PII, secrets, credentials, and
sensitive data. If findings are reported, stop and present them to the user. Do not
push code that contains PII or secrets.

---

## Phase 5 — Review Conventional Commits

Review the commit history on this branch to ensure it tells a clear story and follows
[Conventional Commits](https://www.conventionalcommits.org/) format. This matters
because release tooling (Release Please, semantic-release) uses these prefixes to
determine version bumps and generate changelogs — and because good commit history is
documentation for your future self.

### Step 1: Assess the current commits

List all commits on this branch since it diverged from the base branch:

```bash
git log --oneline <base-branch>..HEAD
```

Evaluate each commit against these criteria:
- **Prefix**: does it have a conventional prefix (`feat:`, `fix:`, `docs:`, etc.)?
- **Subject**: does it explain *why* the change was made, not just *what* changed?
  Bad: `fix: update parser` — what was wrong? why?
  Good: `fix: handle quoted delimiters in CSV fields` — now you know.
- **Scope**: are commits focused on one logical change, or do they bundle unrelated work?
- **Noise**: are there WIP commits, fixups, or "oops" commits that add nothing to history?

### Step 2: Decide what to do

**If commits are already clean** — good prefixes, meaningful messages, focused changes —
confirm they look good and move on. Don't touch what doesn't need touching.

**If commits need improvement**, present your assessment to the user and propose a plan.
The options range from light to heavy:

1. **Reword only** — keep the same commits, just fix the messages to add conventional
   prefixes and better descriptions. Least disruptive.
2. **Squash noise** — collapse WIP/fixup/oops commits into their parent commits while
   keeping meaningful commits separate. Moderate cleanup.
3. **Full restructure** — reorganize commits into logical units (e.g., separate a commit
   that mixes a feature and a refactor into two). Most disruptive, rarely needed.

Always default to the lightest touch that gets the job done. Some people don't want
their commit history rewritten at all — respect that. If the user declines rewriting,
just ensure any *new* commits going forward use conventional format.

### Conventional commit reference

- `feat:` — new feature (triggers minor version bump)
- `fix:` — bug fix (triggers patch bump)
- `feat!:` or `fix!:` or `BREAKING CHANGE:` footer — breaking change (triggers major bump)
- `docs:`, `chore:`, `refactor:`, `test:`, `ci:` — non-release changes

### What makes a good commit message

The subject line should complete the sentence "This commit will..." and capture the
*why*, not just the *what*:

```
feat: add webhook retry with exponential backoff

Webhooks were silently dropped when the target server returned 5xx.
Users reported missing notifications with no way to diagnose. This adds
retry with exponential backoff (3 attempts, 1s/5s/25s) and logs each
attempt so failures are visible in the audit trail.
```

The description is optional for small, self-explanatory changes — but for anything
non-trivial, a sentence or two explaining the motivation makes the commit history
genuinely useful months later.

---

## Phase 6 — Push

Push the branch to the remote:

```bash
git push -u origin HEAD
```

If the branch was rebased in Phase 5, a force push may be needed. Confirm with the
user before force-pushing: "The commits were rebased — I need to force-push. OK?"

---

## Phase 7 — Pull Request

**Only if this is a GitHub/GitLab project and there isn't already an open PR.**

Create a pull request:

```bash
gh pr create --title "<conventional-style title>" --body "$(cat <<'EOF'
## Summary
<1-3 bullet points summarizing the change>

## Test plan
<what was tested, coverage stats>

## Related issues
<Closes #X if applicable>
EOF
)"
```

Include `Closes #X` in the PR body if there's a related issue — this auto-closes the
issue when the PR merges.

**Approval**: based on the user's preference from Phase 1:
- If self-approve: `gh pr review --approve` then `gh pr merge --auto`
- If user approves: tell them the PR is ready and provide the URL

---

## Phase 8 — Wait for Release (if applicable)

**Only if the project has a release pipeline (Release Please, semantic-release, etc.)**

After the PR merges:

1. Watch for the release pipeline to trigger:
   ```bash
   gh run list --branch main --limit 5
   ```
2. If Release Please is configured, it will create a release PR. Watch for it:
   ```bash
   gh pr list --label "autorelease: pending"
   ```
3. Once the release PR appears, let the user know. If they've authorized self-approval,
   approve and merge it. Otherwise, prompt them.
4. After the release PR merges, watch for the release to be minted:
   ```bash
   gh release list --limit 3
   ```
5. Confirm the release exists and has the expected version and artifacts.

If any pipeline step fails, check the run logs (`gh run view <id> --log-failed`) and
report the failure to the user.

---

## Phase 9 — Install & Validate

**Only if a release was minted in Phase 8.**

Download and install the release artifact locally:

1. **Check for install scripts first**:
   - If `scripts/install.sh` exists (and you're on Linux/macOS or WSL):
     ```bash
     bash scripts/install.sh
     ```
   - If `scripts/install.ps1` exists (and you're on Windows):
     ```powershell
     .\scripts\install.ps1
     ```
   - If a Taskfile has an `install` target: `task install`

2. **If no install script**, download the release artifact directly:
   ```bash
   gh release download <tag> --pattern "*$(uname -s)*$(uname -m)*" --dir /tmp
   ```
   Extract and install to a location on PATH.

3. **Validate the installation**:
   - Run the binary with `--version` to confirm the new version
   - Run a quick smoke test relevant to the feature or fix that was implemented
   - If the work was a bug fix, reproduce the original scenario and confirm it's fixed
   - If the work was a new feature, exercise the feature end-to-end

Report the validation results to the user.

---

## Phase 10 — Close Issues

**Only if related issues were identified in Phase 0.**

If the PR included `Closes #X`, the issue should auto-close on merge. Verify:

```bash
gh issue view <number> --json state
```

If the issue is still open (or if `Closes` wasn't in the PR body), close it manually
with a summary comment:

```bash
gh issue close <number> --comment "$(cat <<'EOF'
Resolved in <PR link>.

<Brief summary of what was done and how it addresses the issue.>
EOF
)"
```

---

## Completion

Present a final summary:

```
Ship-it complete:
  Tests:     ✓ passed (XX% coverage)
  Review:    ✓ no blocking issues
  PII scan:  ✓ clean
  Commits:   ✓ conventional format
  Push:      ✓ <branch>
  PR:        ✓ <PR URL>
  Release:   ✓ vX.Y.Z (or N/A)
  Install:   ✓ validated (or N/A)
  Issues:    ✓ #X closed (or N/A)
```

---

## Adapting to What's Available

Not every project has every piece of this pipeline. The skill gracefully degrades:

- **No test suite?** Skip Phase 2, but warn the user there are no tests.
- **No release pipeline?** Stop after Phase 7 (PR creation). The work ships when the PR merges.
- **No install scripts?** Try `gh release download` or `go install` / `npm install -g` / `cargo install` as appropriate.
- **No GitHub?** Skip PR, release, and issue phases. Just validate, commit, and push.
- **No related issues?** Skip Phase 10.
- **Already on main?** Skip PR creation, just push directly (but confirm with the user first — pushing directly to main is unusual).
