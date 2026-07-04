# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`braindump` is a Go CLI that reads AI agent session histories from local disk (Claude Code and Goose AI) and emits them in a single unified JSON schema for analysis, archival, or programmatic processing. Pure Go, no CGo (SQLite via `modernc.org/sqlite`).

## Commands

The `Justfile` is the source of truth for dev tasks. `go` and `just` are provided via a Hermit environment in `bin/` (run `source bin/activate-hermit` if the tools aren't on your PATH).

- Build: `just build` (→ `./braindump`)
- Run all tests: `just test` (or `go test ./...`)
- Run a single package's tests: `go test ./internal/claude/`
- Run a single test: `go test ./internal/claude/ -run TestName`
- Verbose / coverage: `just test-verbose`, `just test-coverage`, `just coverage` (HTML report)
- Lint: `just lint` (`golangci-lint run ./...`)
- Full pre-flight (matches CI intent): `just check` (= fmt-check + vet + lint + test)
- Run the tool locally: `just run -- --pretty` or `go run ./cmd/braindump --pretty`

`lefthook` git hooks run `just fmt` on pre-commit and `just test` + `just lint` on pre-push.

## Architecture

The pipeline in `cmd/braindump/main.go` is: **read all sources → merge → filter → write**. Each agent has an independent reader; `main.go` calls them conditionally based on `--agent`, concatenates the results into `[]model.Session`, then applies filters and picks an output format.

**`internal/model/types.go` is the contract.** Every reader's job is to translate its agent's native storage format into these shared structs (`Session` → `Message` → `ContentBlock`, plus `SessionMetadata`, `MessageMetadata`, `Subagent`). When adding a new agent or field, this file is the anchor — all readers and both output writers depend on it. Agent-specific fields that don't map cleanly go into the `Extra map[string]string` bags rather than new top-level fields.

**Readers are the per-agent boundary** (`internal/claude`, `internal/goose`), each split into `reader.go` (locates + iterates storage) and `parser.go` (maps raw records → `model` structs):
- **Claude** reads JSONL from `~/.claude/projects/*/`. `reader.go` walks the tree, skips any path containing `subagents`, derives session-level metadata from the first line, and computes `created_at`/`updated_at` as the min/max message timestamp. Subagents are loaded separately from `<sessionDir>/<sessionID>/subagents/agent-*.jsonl`. `parser.go` handles Claude's polymorphic `content` field (either a plain string or an array of typed blocks).
- **Goose** reads SQLite from `~/.local/share/goose/sessions/sessions.db` via two queries (`sessions`, then `messages` per session). Model name is dug out of the `model_config_json` blob; leftover columns land in `Extra`.

**Design principle — degrade, don't fail.** A missing source directory/DB returns an empty slice (not an error), and malformed lines / unreadable individual sessions/subagents are logged to stderr with a `Warning:` prefix and skipped. Preserve this: never let one bad record abort the whole dump.

**`internal/filter`** applies `--agent`/`--session-id`/`--since`/`--until` uniformly across the merged `[]model.Session`, after readers run — filtering is format-agnostic and works on the unified model, not on raw agent data.

**`internal/output`** has two writers over the same `[]model.Session`: `writer.go` (JSON, honoring `--pretty`) and `summary.go` (`--summary`, human-readable).

## Release

Tagging and releasing are automated via GitHub Actions. The `tag.yml` workflow (manual dispatch) bumps semver and pushes a `v*` tag; pushing that tag triggers `release.yml`, which runs GoReleaser (`.goreleaser.yml`) to cross-compile for linux/darwin/windows × amd64/arm64. Releases are driven by tags — do not hand-build release artifacts.
