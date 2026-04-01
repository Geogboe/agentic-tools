---
name: go-devops
description: >
  Sets up a complete DevOps pipeline for a Go-based CLI project: CI quality gates
  (golangci-lint v2, markdownlint, yamllint, govulncheck, gitleaks), automated releases
  (Release Please + GoReleaser via the github-release-expert skill), one-command installer
  scripts (via the github-release-installers skill), a go generate templating pattern to
  eliminate duplication, a standard Taskfile, and PII scanning before every git push (via
  the repo-pii-scanner skill). Use when asked to set up DevOps for a Go CLI, automate
  releases, add linting pipelines, add installer scripts, or "make this a real tool."
---

# Go CLI DevOps Pipeline

You are setting up a complete, production-grade DevOps pipeline for a Go CLI project. Work
through the phases in order. Each phase must be fully complete before moving to the next.

## Referencing other skills

This skill orchestrates three other skills. Invoke them at the phases described below:

- **`repo-pii-scanner`** — invoke before every `git push` in this workflow
- **`github-release-expert`** — invoke during Phase 4 (Release Pipeline)
- **`github-release-installers`** — invoke during Phase 5 (Installer Scripts)

To invoke a skill, use the `skill` tool with the skill name.

---

## Phase 1 — Discover

Audit the repository before making any changes. Answer:

1. **Module path** — read `go.mod` for `module` and `go` version
2. **Binary name** — find `cmd/<name>/main.go` or equivalent entry point
3. **Existing CI** — list `.github/workflows/*.yml` and summarize what each does
4. **Existing release setup** — check for `release-please-config.json`, `.goreleaser.yml`, tags
5. **Existing Taskfile** — read `Taskfile.yml` if present; note existing tasks
6. **Install scripts** — check for `scripts/install.sh`, `scripts/install.ps1`
7. **Linter configs** — check for `.golangci.yml`, `.markdownlint.json`, `.yamllint.yml`

Present a brief audit table to the user, then proceed.

---

## Phase 2 — CI Quality Gates

Create or update `.github/workflows/ci.yml`. Trigger on `push` and `pull_request` to the
default branch. Use separate jobs so failures are isolated:

### Job: lint (Go)

```yaml
lint:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v5
    - uses: actions/setup-go@v6
      with:
        go-version-file: go.mod
    - name: Install golangci-lint
      run: go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@latest
    - name: Run golangci-lint
      run: golangci-lint run ./...
```

**Critical:** always use `golangci-lint/v2/cmd/golangci-lint` (v2 module path). The v1 path
silently installs v1 which rejects v2 config files. Never use `golangci-lint-action` with
`version: latest` — pre-built binaries lag behind Go releases and fail on newer `go.mod` targets.

Create `.golangci.yml` if absent:

```yaml
version: "2"

linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - gosec
    - unused
    - gocritic
    - misspell

  settings:
    gosec:
      excludes:
        - G204   # subprocess via variable (intentional in CLI tools)
        - G304   # file inclusion via variable (intentional in file loaders)

  exclusions:
    rules:
      - path: _test\.go
        linters:
          - errcheck
          - gosec

formatters:
  enable:
    - gofmt
```

### Job: lint-markdown

```yaml
lint-markdown:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v5
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
    - run: npm install -g markdownlint-cli
    - run: markdownlint '**/*.md' --ignore node_modules
```

Create `.markdownlint.json` if absent:

```json
{
  "default": true,
  "MD013": false,
  "MD033": false,
  "MD041": false
}
```

### Job: lint-yaml

```yaml
lint-yaml:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v5
    - run: pip install yamllint
    - run: yamllint .
```

Create `.yamllint.yml` if absent:

```yaml
extends: default
rules:
  line-length:
    max: 120
  truthy:
    allowed-values: ['true', 'false']
```

### Job: security

```yaml
security:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v5
      with:
        fetch-depth: 0
    - uses: actions/setup-go@v6
      with:
        go-version-file: go.mod
    - name: Install gitleaks
      run: |
        gh release download --repo gitleaks/gitleaks \
          --pattern "*linux_x64.tar.gz" -O - | tar -xz -C /usr/local/bin gitleaks
      env:
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    - name: Run gitleaks
      run: gitleaks detect --source=. --verbose --redact
    - name: Install govulncheck
      run: go install golang.org/x/vuln/cmd/govulncheck@latest
    - name: Run govulncheck
      run: govulncheck ./...
```

**Note:** Use gitleaks CLI (not `gitleaks-action`) — the Action requires a paid license for
private repos. The CLI works everywhere with just `GITHUB_TOKEN`.

### Job: build

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v5
    - uses: actions/setup-go@v6
      with:
        go-version-file: go.mod
    - run: go build ./...
```

### Job: test

```yaml
test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v5
    - uses: actions/setup-go@v6
      with:
        go-version-file: go.mod
    - run: go test ./...
```

---

## Phase 3 — Taskfile

Create or update `Taskfile.yml` to mirror CI checks locally. Standard task set:

```yaml
# yaml-language-server: $schema=https://taskfile.dev/schema.json
version: '3'

tasks:
  default: task --list

  build:
    desc: Build local binary
    cmds:
      - go build -o ./<binary> ./cmd/<binary>

  install:
    desc: Install binary into GOBIN
    cmds:
      - go install ./cmd/<binary>

  test:
    desc: Run all Go tests
    aliases: [t]
    cmds:
      - go test ./...

  fmt:
    desc: Format all Go source files
    aliases: [format]
    cmds:
      - gofmt -w .

  lint:
    desc: Run golangci-lint (same as CI)
    cmds:
      - golangci-lint run ./...

  lint:md:
    desc: Lint markdown files
    cmds:
      - markdownlint '**/*.md' --ignore node_modules

  lint:yaml:
    desc: Lint YAML files
    cmds:
      - yamllint .

  vuln:
    desc: Run govulncheck
    cmds:
      - govulncheck ./...

  secrets:
    desc: Run gitleaks secrets scan
    cmds:
      - gitleaks detect --source=. --verbose --redact

  generate:
    desc: Run go generate (regenerate derived files from templates)
    aliases: [gen]
    cmds:
      - go generate ./...

  ci:
    desc: Run all CI checks locally (secrets → lint → vuln → test → build)
    cmds:
      - task: secrets
      - task: lint
      - task: lint:md
      - task: lint:yaml
      - task: vuln
      - task: test
      - task: build

  release:check:
    desc: Validate GoReleaser config
    cmds:
      - go run -modfile=tools/go.mod github.com/goreleaser/goreleaser/v2 check --config .goreleaser.yml

  release:snapshot:
    desc: Run a local GoReleaser snapshot build
    cmds:
      - go run -modfile=tools/go.mod github.com/goreleaser/goreleaser/v2 release --snapshot --clean --config .goreleaser.yml
```

Replace `<binary>` with the actual binary name found in Phase 1.

---

## Phase 4 — Release Pipeline

**Invoke the `github-release-expert` skill now.**

That skill will:
- Set up `release-please-config.json` and `.release-please-manifest.json`
- Create `.github/workflows/release.yml` using the two-job pattern (release-please + goreleaser in the same file — never split them)
- Create `.goreleaser.yml` with cross-compilation for linux/darwin/windows × amd64/arm64
- Configure repository Actions permissions
- Validate end-to-end with a test push

**After the skill completes**, verify `.goreleaser.yml` includes:
- All target platforms (no unnecessary `ignore:` entries unless there's a CGO reason)
- `checksums.txt` name template (required by installer scripts)
- SBOM generation with syft

---

## Phase 5 — Installer Scripts

**Invoke the `github-release-installers` skill now.**

That skill will create `scripts/install.sh` and `scripts/install.ps1` with:
- Env-var-only interface (`<BINARY>_VERSION`, `<BINARY>_INSTALL_DIR`, `<BINARY>_FORCE`, `<BINARY>_DEBUG`)
- SHA256 checksum verification against `checksums.txt`
- OS/arch detection and platform-specific asset naming
- PATH management and post-install verification

### Go Generate Pattern (recommended for Go projects)

After the installer scripts are created, eliminate duplication by templating them from a
shared Go constants package. This keeps repo name, binary name, API URLs, and asset naming
in sync automatically.

**Step A** — Create `internal/buildcfg/buildcfg.go`:

```go
// Package buildcfg defines project-level constants shared by the install
// script generator and the self-update command.
package buildcfg

const Repo = "owner/repo"           // GitHub owner/name
const BinaryName = "mybinary"       // compiled binary name
const DefaultInstallDir = ".local/bin"
const APIBase = "https://api.github.com"
const DownloadBase = "https://github.com"

// AssetName returns the archive base name matching the GoReleaser name_template.
func AssetName(version, goos, goarch string) string {
    return BinaryName + "_" + version + "_" + goos + "_" + goarch
}
```

**Step B** — Move installer logic to templates in `scripts/generate/`:
- `install.sh.tmpl` — shell installer with `{{.Repo}}`, `{{.BinaryName}}`, etc.
- `install.ps1.tmpl` — PowerShell installer with same placeholders

**Step C** — Create `scripts/generate/main.go` — a generator that reads `buildcfg`,
renders the templates, and writes `scripts/install.sh` and `scripts/install.ps1`. Normalize
output to LF so shell scripts work on Unix regardless of the host OS.

**Step D** — Create `scripts/doc.go` with the go:generate directive:

```go
//go:generate go run ./generate
package scripts
```

**Step E** — Add a staleness test in `scripts/generate_test.go` that re-runs `go generate`
and fails CI if the committed scripts are out of sync with the templates. Normalize line
endings in the comparison (`bytes.ReplaceAll(b, []byte("\r\n"), []byte("\n"))`) so the test
passes on both Windows and Linux CI runners.

**Step F** — Add to `.gitattributes`:

```
*.sh      text eol=lf
*.sh.tmpl text eol=lf
*.ps1.tmpl text eol=lf
scripts/install.ps1 text eol=lf
```

**Step G** — Add `task generate` to Taskfile (already included in Phase 3 template above).

---

## Phase 6 — PII Gate

**Invoke the `repo-pii-scanner` skill before every `git push` in this workflow.**

This includes:
- After Phase 2 (before pushing CI workflow files)
- After Phase 4 (before pushing release config)
- After Phase 5 (before pushing installer scripts)
- Any time the user asks to commit and push

The scanner checks both current files and full git history for names, emails, IP addresses,
machine paths, and secrets. Do not push until the user has reviewed and cleared all findings.

Also add `.copilot/` and `.claude/` to `.gitignore` to prevent session state from being
committed accidentally.

---

## Phase 7 — Validate End-to-End

After all phases complete:

1. Run `task ci` locally — must be fully green
2. Push to main — confirm all CI jobs pass in GitHub Actions
3. Push a `feat:` commit — confirm Release Please creates a release PR
4. Merge the release PR — confirm GoReleaser attaches artifacts to the GitHub Release
5. Test installer scripts against the new release:
   - Linux/macOS: `curl -fsSL .../install.sh | sh`
   - Windows: `irm .../install.ps1 | iex`
6. If the binary has a self-update command, verify it resolves the correct asset for the current platform

---

## Common Pitfalls Reference

| Problem | Fix |
|---|---|
| golangci-lint version mismatch with go.mod | `go install golangci-lint/v2/cmd/golangci-lint@latest` (always build from source) |
| golangci-lint v1 vs v2 config rejected | Use `golangci-lint/v2/cmd/golangci-lint` module path (not v1) |
| GoReleaser assets not attached | Release Please and GoReleaser must be in the same workflow file (two-job pattern) |
| GoReleaser "tag not against commit" | Use `ref: ${{ needs.release-please.outputs.tag_name }}` in checkout |
| GoReleaser SBOM fails (syft not found) | Add `anchore/sbom-action/download-syft@v0` step before goreleaser |
| gitleaks license error (private repo) | Use CLI via `gh release download`, not `gitleaks-action` |
| install.ps1 CRLF breaks on Linux CI | Add `scripts/install.ps1 text eol=lf` to `.gitattributes`; normalize in staleness test |
| go generate CRLF on Windows host | Normalize `bytes.ReplaceAll(buf, []byte("\r\n"), []byte("\n"))` in generator |
| `.copilot/` committed accidentally | Add `.copilot/` to `.gitignore` before first push |
