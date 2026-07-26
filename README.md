# artifact-graph

[中文](README.zh-CN.md)

Git-native Markdown artifact graph scanner and validator for agentic coding workflows.

`artifact-graph` helps projects keep requirements, scenarios, design notes, source files, tests,
and version-lock metadata connected. It is designed for deterministic local use before an AI coding
agent claims implementation work is complete.

<!-- release-skill:capability:external-write-boundary -->
> **External-write boundary:** Installing `artifact-graph` does not modify a project or write to
> remote systems. Read-only commands such as `--help`, `doctor`, `validate`, `query`, and `audit`
> do not change project files. Commands including `init`, `version-lock refresh`, and
> `hooks install-git` write locally only when explicitly invoked.

<!-- release-skill:capability:safe-first-command -->
> **Safe first command:** Inspect the installed CLI and project health before authorizing any
> initialization, version-lock refresh, or hook installation.

```bash
pnpm exec artifact-graph --help
pnpm exec artifact-graph doctor --root .
```

If either read-only command fails, confirm that Node.js `>=22.0.0` is active and reinstall the
package using the precise instructions in [INSTALL.md](INSTALL.md). Do not run a write command
until the CLI resolves successfully and doctor reports an actionable diagnosis.

## Install

```bash
pnpm add -D artifact-graph
```

Or install from GitHub:

```bash
npm install --save-dev github:ifoohoo/artifact-graph
```

Node.js `>=22.0.0` is required. For pnpm 10+, see [INSTALL.md](INSTALL.md) for the native build
allowlist setup.

## Quick Start

```bash
# pnpm
pnpm exec artifact-graph init --root .
pnpm exec artifact-graph validate --root . --warning-only
pnpm exec artifact-graph version-lock refresh --all --format markdown
pnpm exec artifact-graph version-lock audit --root . --strict-missing-lock

# npm
npx artifact-graph init --root .
npx artifact-graph validate --root . --warning-only
npx artifact-graph version-lock refresh --all --format markdown
npx artifact-graph version-lock audit --root . --strict-missing-lock
```

> Use `version-lock refresh --all` for the initial lock. The `--changed-only --staged` variant is for
> pre-commit hooks on existing projects — not for first-time initialization.

## Common Workflows

- Generate or inspect project artifact graph configuration with `artifact-graph init`.
- Validate artifact links with `artifact-graph validate`.
- Validate Review Result Protocol v1.0 documents with `artifact-graph validate-review-result --file <path>`.
- Build implementation context with `artifact-graph context` or `artifact-graph packet`.
- Keep traceability freshness with `artifact-graph version-lock refresh` and `audit`.
- Version lock covers both implementation/verification edges (`locks`) and artifact-to-artifact
  relations (`artifactRelations`). Old 1.0 lock files without `artifactRelations` are treated as
  having an empty relation list; run `refresh --all` once to establish the complete relation baseline.
- Install opt-in Git hooks with `artifact-graph hooks install-git --hook all`.

## Review Result Protocol

The package publishes `schemas/review-result.schema.json` and a matching TypeScript types + validateReviewResult validator API.
The protocol is project-neutral and supports review, repair, batch evidence, findings, metrics, and
fail-closed decisions. Unknown top-level fields are rejected; `attempt` is limited to 1–3;
successful acceptance requires `producer`; and `PASS`/`PASS_WITH_RESIDUAL_MINOR` cannot contain an
open `block` finding. Independent repair re-review can record `acceptance.reviewer` and
`acceptance.source_result`; the validator rejects self-acceptance by the repair producer. Invalid
fields and semantic violations are reported with stable JSON paths. JSON Schema cannot compare
cross-object field values, so callers must also run the semantic validator; stable identity is
`executor + name`, while `skill` is only metadata and cannot establish independence.

## Related Project

Use [`artifact-chain-assistant`](https://github.com/ifoohoo/artifact-chain-assistant) for Codex and
Claude Code skills that guide artifact-chain intake, setup, and maintenance.

## License

Apache-2.0. See [LICENSE](LICENSE).
