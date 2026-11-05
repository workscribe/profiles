# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A static JSON data registry — no build, lint, or test tooling. It is `git@github.com:workscribe/profiles.git`, and `workscribe-cli` fetches files from it directly via GitHub raw URLs at runtime (`https://raw.githubusercontent.com/workscribe/profiles/main/<name>.json`), not as a package dependency. There is no local dev loop; changes take effect for CLI users as soon as they're pushed to `main`.

Each profile adds language/framework-specific command-normalization rules on top of the CLI's built-in rules in `workscribe-cli/src/normalize/patterns.ts`, so `workscribe summary` can categorize commands like `poetry add` or `cargo clippy` correctly.

## Repo structure

- `manifest.json` — the index. `workscribe-cli` fetches this first to list installable profiles and to auto-detect which ones apply to a project (via `detect.files`).
- `<name>.json` (e.g. `python.json`, `go.json`, `developer.json`) — one file per profile, fetched on demand when a user runs `workscribe profile add <name>`.

## Schema (validated by workscribe-cli, not by anything in this repo)

`workscribe-cli/src/profiles/schema.ts` is the source of truth — `isValidProfile`/`isValidManifest` run client-side after fetch, and the CLI silently skips anything that fails validation (no error surfaced to the registry). Check that file before changing shape here.

**Manifest entry** (`manifest.json`, array):
```
{ name, displayName, tags: string[], description, keywords: string[], detect?: { files: string[] } }
```
Only `name` (string) is actually enforced by `isValidManifest` — the rest is convention.

**Profile file** (`<name>.json`):
```
{
  name, displayName, version,           // all required non-empty strings
  detect?: { files: string[] },         // project files that trigger auto-suggestion
  rules: [{ pattern, category, metadata?: Record<string,string> }]   // pattern/category required
}
```
- `pattern` is a JS regex source string (anchored with `^` in every existing profile). Invalid regex or a rule missing `pattern`/`category` is dropped at load time — the whole profile still loads, just minus that rule.
- `metadata` values support `$1`, `$2`, … backreferences, substituted from the regex match (see `alembic` rule in `python.json` for an example: `"subcommand": "$1"`).
- `category` should match the category vocabulary the CLI already uses (`dependencies`, `testing`, `building`, `running`, `linting`, `exploring`, …) — check `workscribe-cli/src/normalize/patterns.ts` for the existing set before inventing a new one.

## Adding or editing a profile

1. Filename stem must equal the profile's `name` field (`rust.json` → `"name": "rust"`) — the CLI requests `<name>.json` by that name.
2. Add or update the corresponding entry in `manifest.json` — a profile with no manifest entry can still be fetched directly but won't be discoverable or auto-detected.
3. Use `detect.files` sparingly — it should be a file whose presence reliably implies the language/tool (`Cargo.toml`, `go.mod`, `Gemfile`), not something generic.
4. Rules are not ordered/prioritized against each other within the CLI the way `patterns.ts` rules are — each profile's rules are simply appended to the active rule set — so keep individual patterns specific enough not to over-match.
5. Commit style in this repo follows Conventional Commits (`feat:`, `fix:`) per existing history.
