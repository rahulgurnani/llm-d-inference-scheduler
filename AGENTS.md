# Agent Instructions for llm-d-inference-scheduler

This file provides shared guidance for AI coding assistants (Claude Code, Gemini CLI,
GitHub Copilot, Cursor, and others) working in this repository. Instructions here
reflect team conventions; follow them to avoid avoidable review comments.

> **Deeper references** — load on demand when relevant:
> - Build & test details: `DEVELOPMENT.md`
> - Architecture overview: `README.md`
> - Disaggregation design: `docs/disaggregation.md`

---

## Project Overview

This repository builds the **Endpoint Picker Plugin (EPP)**, the inference scheduling
component that routes LLM requests to vLLM backends. The EPP runs alongside a
Gateway API implementation and picks backends based on KV-cache state, prefill
locality, and load.

Key packages:
- `cmd/epp/` — EPP binary entrypoint
- `pkg/epp/requestcontrol/` — request lifecycle orchestration (Director)
- `pkg/epp/framework/` — plugin interfaces and built-in plugins
- `pkg/epp/scheduling/` — backend selection logic
- `pkg/epp/datalayer/` — data graph and shared state
- `pkg/epp/flowcontrol/` — admission and fairness
- `apix/` — CRD API types
- `test/` — integration and e2e test suites

All Go modules live under `github.com/llm-d/llm-d-inference-scheduler`.

---

## Before You Push

Run the full presubmit check locally before opening a PR:

```bash
make presubmit
```

This runs (in order): branch check, signed-commit check, `go mod tidy` check,
`gofmt` + `golangci-lint fmt`, `golangci-lint run`, `typos`, and `govulncheck`.
CI runs the same checks — fix locally first.

---

## Build & Test Commands

```bash
# Build
make image-build-epp          # build EPP container image
make image-build-sidecar      # build routing sidecar image

# Unit tests (race detection + coverage always enabled)
make test-unit                 # all unit tests (epp + sidecar)
make test-unit-epp             # epp only
make test-unit-sidecar         # sidecar only
make test-filter PATTERN=Foo   # run tests matching name pattern

# Integration tests (requires a running kind cluster)
make test-integration

# Hermetic integration tests (no external cluster needed)
make test-integration-hermetic

# End-to-end tests (spins up a temporary kind cluster)
make test-e2e

# Local dev cluster
make env-dev-kind              # create/refresh kind cluster + deploy
make clean-env-dev-kind        # tear down

# Code quality
make format                    # gofmt + golangci-lint fmt
make lint                      # golangci-lint run
make vulncheck                 # govulncheck
```

---

## Code Conventions

### File headers

Every new `.go` file must start with the Apache 2.0 license header:

```go
/*
Copyright 2026 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
...
*/
```

Copy the header from any existing file in the repo.

### Import grouping

Three groups, separated by blank lines, in this order:

```go
import (
    // 1. stdlib
    "context"
    "fmt"

    // 2. external modules
    "go.opentelemetry.io/otel"
    "sigs.k8s.io/controller-runtime/pkg/log"

    // 3. project-local
    "github.com/llm-d/llm-d-inference-scheduler/pkg/epp/datalayer"
)
```

`gofmt` and `golangci-lint fmt` enforce this — run `make format` before committing.

### Error handling

- Wrap errors with context using `fmt.Errorf("doing X: %w", err)`.
- Use the project's `errcommon.Error` struct (in `pkg/common/error/`) for errors
  that need a gRPC/HTTP status code at the boundary.
- Do not swallow errors silently; log or return them.

### Linting

- Never add `//nolint` without a justifying comment explaining why the lint rule
  does not apply in this specific case.
- All lint issues introduced in your diff must be fixed before merging.

### Generated files

Do **not** manually edit files matching `zz_generated.*`. Re-run the relevant
generator (`make generate` or the specific `//go:generate` directive) instead.

---

## Testing Practices

### Unit tests

- File: `foo_test.go` in the **same package** as the code under test (white-box).
- Framework: stdlib `testing` + `github.com/stretchr/testify/assert` (or `require`).
- Race detection and coverage are always on (`-race -coverprofile`).
- Place fixture data in a `testdata/` subdirectory next to the test file.

### End-to-end tests

- Location: `test/e2e/`
- Framework: **Ginkgo v2 / Gomega** (`github.com/onsi/ginkgo/v2`, `github.com/onsi/gomega`).
- E2E tests spin up their own kind cluster; do not assume a pre-existing cluster.

### Integration tests

- Location: `test/integration/`
- Tagged with `//go:build integration_tests`.
- Hermetic variants use `controller-runtime/pkg/envtest` (no external cluster).

---

## PR and Commit Conventions

### Commit messages

- Subject line: `<type>(<scope>): <short summary>` — e.g., `fix(requestcontrol): handle nil endpoint`.
- Use the imperative mood: "fix", "add", "remove", not "fixed" or "adds".
- Reference the issue in the PR description (`Fixes #<number>`), not in every commit.
- Commits **must be signed** (`git commit -s`). The `signed-commits-check` target
  in `make presubmit` enforces this against `upstream/main`.

### Do NOT add AI coauthor lines

Do not append `Co-authored-by:` or `Co-Authored-By:` trailers for AI assistants
to commit messages. This is a firm project policy.

### PR description

Use the template in `.github/PULL_REQUEST_TEMPLATE.md`. Always fill in:
- `/kind` label (`bug`, `cleanup`, `documentation`, `feature`, `test`)
- What the PR does and why
- Which issue it fixes (`Fixes #<number>`)
- Release note (or `NONE`)

### Branch policy

Do not commit directly to `main`. Always work on a feature branch and open a PR.
`make presubmit` will fail if `HEAD` is on `main`.

---

## What Not to Touch

- `zz_generated.*` files — generated; edit the source and re-run the generator.
- `vendor/` — managed by `go mod vendor`; do not hand-edit.
- `.github/workflows/` — CI configuration; changes require explicit discussion.
- `go.mod` / `go.sum` — run `go mod tidy` (or `make go-mod-check`) rather than
  editing manually.

---

## Agent Scratch Space

Store ephemeral work files (research notes, plans, draft specs) under
`.agent-workspace/` at the repo root. This path is gitignored and will not appear
in PRs. Do not commit files from this directory.
