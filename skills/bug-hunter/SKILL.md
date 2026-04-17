---
name: bug-hunter
description: >
  Systematically finds and captures bugs in a project by using it the way a real user would —
  reading the docs, following the happy path, then probing robustness with awkward inputs,
  realistic workflow variations, edge cases, and natural-limit pushes. Files findings as
  GitHub/GitLab issues or a markdown doc.
  Use this skill whenever the user says "find bugs", "hunt for bugs", "stress test this",
  "QA this tool", "what's broken", "interactively test", "find and capture bugs", or similar.
  Works on anything: CLI binaries, REST APIs, background daemons, Docker services, web apps, libraries.
---

# Bug Hunter

Act like a curious, practical first-time user trying to get real work done. Use the tool the way a
real user would, not like a framework author building a big harness. Follow the documented path,
verify what happened in the real backend, then vary the workflow in slightly awkward but realistic
ways to see where the program stops functioning properly.

---

## Step 0: Two quick questions before you start

Only ask these if the answer is not already clear from the user's request or repo context:

1. **Where should findings go?**
   - GitHub issues (`gh issue create`) — only to this repo
   - GitLab issues (`glab issue create`) — only to this repo
   - A markdown file in the repo (e.g. `bugs-found.md`)
   - Just show them in the conversation

2. **Scope** — is there a specific area, workflow, or command to focus on, or is it open-ended?

If the user says "just go for it" or something similarly open-ended, default to: explore the whole
tool, file as GitHub issues if `gh` is available and authenticated, otherwise write a `bugs-found.md`.
If the user already named the tracker, file format, or focus area, do not stop to re-ask it.

---

## Step 1: Orient before you touch anything

Read whatever documentation exists. Look for:

- `README.md` — what the tool does, how to install/run it
- `docs/` directory — architecture, CLI reference, examples
- `examples/` directory — sample configs and workflows
- Help text from the tool itself (`--help`, `-h`, `help <subcommand>`)

Prefer the project's documented entry points over guessed commands:
- `task` / `Taskfile.yml`
- `npm` / `pnpm` / `cargo` / `go` scripts documented in the repo
- `docker compose` files
- repo-owned helper scripts

If the repo has a documented runnable example, sample config, or explicit user workflow, start
there before broad repo-wide validation. A bug hunt is primarily about broken product behavior, not
about exhaustively re-running CI.

Use a **parallel Explore subagent** to read docs and examples while you scan help text yourself.
You're not reading every line — you want to know: what is this supposed to do? what does the happy
path look like? what are the main commands/endpoints/workflows?

Don't start testing until you have a working mental model of what "correct" looks like.
Don't spend your opening moves on `test`, `lint`, full builds, or wide source greps unless:
- the user explicitly asked for those checks
- the documented workflow depends on them
- or you already have a concrete suspicion they will validate

---

## Step 2: Happy path first

Follow the documented happy path end-to-end before looking for problems. If the basic thing doesn't
work, start there. If it does work, you have a baseline to deviate from.

For a **CLI tool**, that might be:
```sh
# Install or build
# Run the main command
# Do the thing it's supposed to do
# Clean up / uninstall
```

For a **Docker-backed service**, that might be:
```sh
# docker run / docker compose up
# Hit the API or use the CLI
# Check that expected side effects happened (containers running, files created, etc.)
# Tear down
```

For a **background daemon + API**, that might be:
```sh
# Start the daemon
# Wait for it to report ready
# Create a resource
# Read it back
# Delete it
# Confirm it's actually gone
```

Verify side effects directly — don't just trust what the tool reports. If it says a container is
running, run `docker ps`. If it says a file was written, check the filesystem.
If the happy path breaks immediately, treat that as a likely finding, capture exact repro evidence,
then continue testing adjacent flows if possible so one blocker does not end the whole bug hunt.

Use broad automated checks as support, not as your primary hunt:
- Prefer `--help`, documented examples, and live workflows before `task test` or `npm test`
- If a test suite fails because of missing network, sandbox restrictions, or unrelated fixture setup,
  record that as a blocker only if it prevents fair workflow testing
- Don't let generic CI noise crowd out confirmed user-visible failures

---

## Step 3: Probe robustness like a user

Now vary the workflow. You're not doing security research or exploit hunting. You're checking
whether the program continues to function properly when a normal user is imperfect, repeats
actions, restarts things, or uses the tool in a slightly awkward order.

Work through these **naturally**, not as a rigid checklist. Skip anything that clearly doesn't apply.

### Give it awkward inputs
- Missing required fields or arguments
- Empty strings, empty arrays, zero counts, negative numbers
- Very long strings, special characters, spaces in names
- Wrong types where the tool accepts freeform input (e.g., `type: badvalue` in a config)
- Invalid references (pool names, IDs, file paths that don't exist)

### Push natural limits
- Create several instances of the same thing — does it honor any stated limits (max, quota, cap)?
- Create two things with the same name
- Delete something that doesn't exist
- Delete the same thing twice
- List/get when nothing exists — does it return a clean empty result or an error?

### Vary the workflow
- Run the happy path once, then retry part of it without fully cleaning up
- Restart the daemon or rerun the command and see what survives
- Run from the wrong directory, then recover and continue
- Mix modes if the tool supports them (standalone vs server, CLI vs API, config file vs simple mode)
- Retry after a partial success and see whether state becomes inconsistent
- Use the tool in a slightly different order than the docs, if that still feels like normal user behavior

### Check what's left behind
After operations, verify the backend matches what the tool reports:
- Leaked processes, containers, temp files after a failed operation
- Resources that weren't cleaned up on delete
- State files that grow without bound

### Test error paths
- Run commands against a service that isn't running
- Use a config file with intentional mistakes
- Interrupt a long-running operation mid-way (Ctrl-C)

### Check the small stuff
- Do exit codes make sense? (`echo $?` — 0 on success, non-zero on real errors, *not* on empty results)
- Do error messages tell you what went wrong and how to fix it?
- Does `--help` text match actual behavior?
- If there's a config validator, does it catch the same things the tool rejects at runtime?

### Keep an evidence trail
- Save the exact command, working directory, and key output for each confirmed bug
- Distinguish product bugs from setup mistakes or unsupported environments
- Re-run from a clean state before filing if stale state could explain the result
- If a limitation is real but not fixable in-session, record it explicitly instead of dropping it
- Treat surprising behavior as a lead, even if it is not yet a confirmed bug

---

## Step 4: Use subagents to move faster

Where tests are **independent** (different commands, different areas, no shared server state),
spawn parallel subagents to run them simultaneously. Good candidates:

- One subagent reads docs/architecture while another explores the CLI surface
- One subagent tests input validation while another tests the REST API
- One subagent checks config edge cases while another tests resource lifecycle

**Don't parallelize** operations that share mutable state (the same running server, the same
local state file, the same Docker containers) — those need to run sequentially to avoid
interference.

When spawning a subagent, give it:
- Exactly what to test
- Where to save its findings (a temp file or stdout summary)
- A reminder to clean up any resources it creates

Prefer smaller subagents for bounded slices of work. Good split examples:
- One subagent: docs + happy-path expectations
- One subagent: CLI/input validation
- One subagent: API or backend cleanup behavior

---

## Step 5: File findings as you go

Don't batch everything until the end — file each finding as soon as you've confirmed it's real
and reproducible. This keeps findings from getting lost if the session ends early.

**Before filing, verify:**
- You can reproduce it from a clean state (right directory, fresh server, no leftover state)
- It's actually a bug and not a test methodology mistake (e.g., working directory drift, stale state file)
- You can explain the user-visible impact in one or two sentences

**Good issue format:**

```
Title: <component>: <short description of what's wrong>

## Summary
One sentence.

## Steps to reproduce
1. ...
2. ...
3. (copy-pasteable commands)

## Expected
What should happen.

## Actual
What actually happens.
```

Keep it short. The goal is something a developer can act on, not a novel.

When using `gh` or `glab`, always target **this repository only** — never file to other repos.
If filing is blocked by auth or permissions, write the issue body to a markdown file instead of dropping the finding.

### When findings go to markdown instead of an issue tracker

Use a consistent report shape so the output is reviewable and comparable across runs:

```md
# Scope
- What you tested
- What environment you used

# Confirmed Findings
## <severity>: <title>
### Summary
### Steps to Reproduce
### Expected
### Actual
### Evidence
### User Impact

# Environment / Setup Blockers
- Things that prevented fair testing but are not confirmed product bugs

# Untested / Not Verified
- Important areas you did not reach
```

Rules:
- Put only confirmed product bugs in `# Confirmed Findings`
- Put Docker daemon problems, missing credentials, unsupported platforms, rate limits, and similar setup issues in `# Environment / Setup Blockers`
- Always include at least one bullet in `# Untested / Not Verified` when the run did not cover the full workflow
- If you tested only a narrow slice, say so explicitly in `# Scope`

---

## Examples by project type

### Go CLI binary

```sh
# Build first if needed
go build -o /tmp/mytool ./cmd/mytool

# Try core commands
/tmp/mytool --help
/tmp/mytool <main-command> --help

# Try the happy path from the docs
/tmp/mytool init
/tmp/mytool do-the-thing

# Poke at it
/tmp/mytool do-the-thing --count -1      # negative count
/tmp/mytool do-the-thing --name ""       # empty name
/tmp/mytool do-the-thing --config /nonexistent/path.yaml
/tmp/mytool some-command; echo "exit: $?"  # exit code on success
/tmp/mytool bogus-command; echo "exit: $?" # exit code on bad input

# Check config validation
echo 'bad: yaml: [' > /tmp/bad.yaml
/tmp/mytool validate --config /tmp/bad.yaml
```

### Docker-backed service

```sh
# Start it
docker compose up -d   # or: ./serve.sh &

# Wait for readiness
sleep 3 && curl -s http://localhost:PORT/healthz

# Happy path
curl -X POST .../resources -d '{...}' | jq .
curl .../resources | jq .

# Verify the backend matches what the API says
docker ps --filter "name=myapp-"

# Stress it
curl -X DELETE .../resources/does-not-exist   # delete nonexistent
curl -X POST .../resources -d '{}'             # missing required fields
for i in $(seq 1 5); do curl -X POST .../resources -d '{"name":"test"}'; done  # duplicates / limits

# Clean up and verify cleanup
curl -X DELETE .../resources/REAL_ID
docker ps --filter "name=myapp-"   # should be gone
```

### Background daemon + REST API

```sh
# Start daemon in background, capture logs
./mydaemon serve --config config.yaml > /tmp/daemon.log 2>&1 &
DAEMON_PID=$!
sleep 3

# Check it started
curl -s http://localhost:PORT/healthz

# Core lifecycle
ID=$(curl -s -X POST .../items -H 'Content-Type: application/json' \
  -d '{"name":"test"}' | jq -r .id)
curl -s .../items/$ID | jq .
curl -s -X DELETE .../items/$ID
curl -s .../items/$ID  # should be 404 now

# Empty list
curl -s .../items | jq .   # should be [] not an error

# Bad inputs via API
curl -s -X POST .../items -d '{"name":""}'  # empty name
curl -s -X POST .../items -d '{}'           # missing name
curl -s .../items/not-a-real-id            # bad ID format

# Kill daemon and check for leftover state
kill $DAEMON_PID
ls /tmp/mydaemon-*    # temp files?
docker ps --filter "name=mydaemon-"   # leaked containers?
```

### Library / SDK

```sh
# Write a small script that uses it in the obvious way
# Then try passing nil/null, empty slices, negative values
# Check whether panics or errors are returned (and whether errors are useful)
# Try calling methods out of expected order
```

---

## Step 6: Wrap up

After you've worked through the tool:

1. **Summarize findings** in the conversation — a quick list of what you filed and what severity you'd assign each (`blocker`, `high`, `medium`, `low`)
2. **Clean up** — kill any background services, remove test containers, delete temp state directories
3. **Note anything you couldn't test** — e.g., "couldn't test the Windows-specific path", "rate limiting requires a paid tier"

If you wrote a `bugs-found.md`, offer to commit it.
