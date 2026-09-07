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

### Code traceability starts with scan scope

Declare links with standalone `// @feature A1` source comments or `<!-- @feature A1 -->`
in skill Markdown. Include those files in `types.test.paths`, which also handles implementation
sources; the default does not scan every `src/` or `skills/` tree. Native Python `#` comments
are not supported.

> `version-lock audit --strict-missing-lock` checks locks for discovered relationships. Files
> without annotations and artifacts without relationships can remain invisible to missing-lock
> checks; even zero locks can pass. It does not prove complete release-file or artifact coverage.

See [Code Traceability And Coverage Boundaries](INSTALL.md#code-traceability-and-coverage-boundaries)
for syntax, scan configuration, classification, exemptions and project isolation. Inspect the
index before `refresh --all`; there is no `version-lock update --all` operation.

### Daily commands

- Generate or inspect project artifact graph configuration with `artifact-graph init`.
- Validate artifact links with `artifact-graph validate`.
- Validate Review Result Protocol v1.0 documents with `artifact-graph validate-review-result --file <path>`.
- Build implementation context with `artifact-graph context` or `artifact-graph packet`.
- Add `--view current|planned|history|all` to `query`, `context`, or `packet` when a project maps
  statuses through `statusViews`. The default stays compatible with the unfiltered graph.
- Inspect change impact without refreshing locks with `artifact-graph impact --worktree`.
- Report graph health, scan mapping, and the limits of behavior/release evidence with
  `artifact-graph coverage`. The command does not infer successful verification or publication.
- Keep traceability freshness with `artifact-graph version-lock refresh` and `audit`.
- Version lock covers both implementation/verification edges (`locks`) and artifact-to-artifact
  relations (`artifactRelations`). Old 1.0 lock files without `artifactRelations` are treated as
  having an empty relation list; run `refresh --all` once to establish the complete relation baseline.
- Install opt-in Git hooks with `artifact-graph hooks install-git --hook all`.

### Cleaning Up Orphan Locks After Deleting or Splitting Artifacts

When an artifact, source file, or traceability edge no longer exists — typically after deleting
or splitting an artifact — its version locks become orphans. `version-lock refresh` retains
orphan locks by default so cleanup stays an explicit, reviewable decision:

```bash
# pnpm
pnpm exec artifact-graph version-lock refresh --all --remove-orphans --format markdown
git diff artifacts/traceability-version-lock.json   # review the removed lock entries
git add artifacts/traceability-version-lock.json    # stage, then commit again

# npm
npx artifact-graph version-lock refresh --all --remove-orphans --format markdown
git diff artifacts/traceability-version-lock.json
git add artifacts/traceability-version-lock.json
```

`version-lock audit` marks structural orphan locks and stale hashes as blocking issues and prints
these same remediation steps. Runner liveness findings (a test file no longer active in any
configured runner) are warnings only and do not block.

## Review Result Protocol

The package publishes `schemas/review-result.schema.json`, plus matching TypeScript types and a
`validateReviewResult` validator API. The protocol is project-neutral: it covers review, repair,
batch evidence, findings, metrics, and fail-closed decisions.

Unknown top-level fields are rejected; `attempt` is limited to 1–3; successful acceptance requires
a `producer`; and `PASS`/`PASS_WITH_RESIDUAL_MINOR` cannot contain an open `block` finding. An
independent repair re-review may record `acceptance.reviewer` and `acceptance.source_result`; the
validator rejects self-acceptance by the repair producer. Invalid fields and semantic violations
are reported with stable JSON paths.

Because JSON Schema cannot compare values across objects, callers must also run the semantic
validator. Stable identity is `executor + name`; `skill` is only metadata and cannot establish
independence.

## Documentation

- [INSTALL.md](INSTALL.md) — detailed installation and pnpm 10+ setup guide
- [CHANGELOG.md](CHANGELOG.md) — release history
- [CONTRIBUTING.md](CONTRIBUTING.md) — contribution guidelines
- [SECURITY.md](SECURITY.md) — security policy

## Related Project

Use [`artifact-chain-assistant`](https://github.com/ifoohoo/artifact-chain-assistant) for Codex and
Claude Code skills that guide artifact-chain intake, setup, and maintenance.

## License

Apache-2.0. See [LICENSE](LICENSE).
