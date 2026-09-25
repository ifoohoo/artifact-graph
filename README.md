# artifact-graph

[中文](README.zh-CN.md)

Git-native artifact graph scanner and validator for agentic coding workflows.
Markdown stays the default parser. A custom type may set `format: json` and bring that original file into the same graph.

`artifact-graph` helps projects keep requirements, scenarios, design notes, source files, tests,
and version-lock metadata connected. It is designed for deterministic local use before an AI coding
agent claims implementation work is complete.

<!-- release-skill:capability:external-write-boundary -->
> **External-write boundary:** Installing `artifact-graph` does not modify a project or write to
> remote systems. Read-only commands such as `--help`, `doctor`, `validate`, `query`, `audit`,
> `check-professional` (without `--conclusion-output`), and `read-proof` do not change project files.
> `--conclusion-output` creates one new proof file at an explicit absolute path and never overwrites
> an existing proof. Commands including `init`, `scan`, `version-lock refresh`, and
> `hooks install-git` write locally only when explicitly invoked.

<!-- release-skill:capability:safe-first-command -->
> **Safe first command:** Inspect the installed CLI and project health before authorizing any
> initialization, version-lock refresh, or hook installation.

```bash
pnpm exec artifact-graph --help
pnpm exec artifact-graph doctor --root .
```

If either read-only command fails, confirm that Node.js `>=22.22.2 <23` is active and reinstall the
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

Node.js `>=22.22.2 <23` is required. For pnpm 10+, see [INSTALL.md](INSTALL.md) for the native build
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

### JSON originals for custom types

Built-in types keep their dedicated parsers. `format: json` belongs only on a custom type registered by the project, and that type must also set `idField`. Omitting `format` keeps Markdown parsing. A `*.json` scan path does not switch parsers until the type declares `format: json`.

`idField` is the name of one property on the root object. The name is used whole, so a dot inside it stays part of the name. Each file must contain one JSON object and produces one node. The id is that property's string, including spaces and case, and is then checked with the type's existing `idPatterns`.

Invalid JSON, a root array or scalar, a missing id, a non-string id, or a blank string produces a diagnostic and no node. A string that fails `idPatterns` still produces the node, together with a diagnostic.

Relations stay in `relationSemantics`, using the existing `kind`, `fields`, `targetTypes`, and `label`. For JSON files, a dotted `fields` entry is a chain of own properties. `capability.capabilityRef` reads the root object's `capability`, then that object's `capabilityRef`. A missing property does not create an optional relation. If an intermediate value is not an object, or the final value is not a non-empty string or an array of non-empty strings, the diagnostic names the original file and the field. Valid array entries use the existing target resolver; a broken target remains a graph validation finding. An empty array creates no edge.

A dot only separates exact property names. `*`, filters, JSONPath, and expressions are not path syntax in this batch. Records inside an array stay in the original object. This batch reads the id and configured relations; it does not select a title or status, and it does not join, prefix, or rewrite ids. Markdown fields are still looked up as one literal string. Diagnostics use line 1 because this batch does not map JSON tokens to source lines.

A project can point a method at a capability and a release record at a method. The configuration below shows those two directions.

```yaml
types:
  capability:
    paths: ["capabilities/*.json"]
    format: json
    idField: capabilityId
  method:
    paths: ["methods/*.json"]
    format: json
    idField: methodId
  skill_release:
    paths: ["releases/*.json"]
    format: json
    idField: releaseId
idPatterns:
  capability: "^CAP-[A-Z0-9-]+$"
  method: "^METH-[A-Z0-9-]+$"
  skill_release: "^REL-[A-Z0-9-]+$"
relationSemantics:
  applies:
    label: Method applies to capability
    targetTypes: [capability]
    fields: [capability.capabilityRef]
  releases:
    label: Release record points to method
    targetTypes: [method]
    fields: [methods]
```

A method file stores one capability id under nested properties. `capability.*` looks up a property named `*` and does not collect the neighboring properties.

```json
{
  "methodId": "METH-1",
  "capability": { "capabilityRef": "CAP-1" }
}
```

A release file stores several method ids in a string array. `method:METH-1` is resolved by the existing `type:id` parser.

```json
{
  "releaseId": "REL-1",
  "methods": ["METH-1", "method:METH-2"]
}
```

`scanArtifacts`, `validate`, and `version-lock` consume the same graph. The lock still hashes the original file bytes. Finding the configured nodes and relations shows that those files entered the graph. Domain review and publication facts need their own evidence.

### Daily commands

- Generate or inspect project artifact graph configuration with `artifact-graph init`.
- Validate artifact links with `artifact-graph validate`.
- Emit a reusable graph professional proof with `artifact-graph check-professional --root . --format json`.
  Add `--conclusion-output <absolute-path>` only when a proof file is required; the command composes
  existing `validate`, `version-lock audit`, and `coverage` checks and does not write graph cache.
- Read one graph proof with `artifact-graph read-proof --proof-root <root> --proof <relative> --format json`.
  `GRAPH_CLEAR` with complete scope is `pass`; findings, incomplete checks, and recorded unavailability
  are `not_pass`. A missing, damaged, foreign, or unknown-code file is `unavailable`. This is not the
  Review Result Protocol.
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
- Inspect and plan artifact restructuring with `artifact-graph restructure inspect` and
  `artifact-graph restructure plan`. These two are read-only compilations of a mapping you supply.

### Restructuring Artifacts (Split, Move, Renumber)

`artifact-graph restructure` compiles an explicit restructuring mapping into a reviewable candidate
plan and then applies the file set as one operation. Three transformations are supported:
`record-split` (physically split records), `identity-split` (split one numbered identity into
several), and `move-renumber` (move records between files and renumber them). Deciding capability
boundaries and where each acceptance criterion goes stays outside the CLI; the compiler only turns
a complete mapping into a plan and applies it.

```bash
# pnpm
pnpm exec artifact-graph restructure inspect --root . --input request.json --format json
pnpm exec artifact-graph restructure plan --root . --input mapping.json --format json
pnpm exec artifact-graph restructure apply --root . --plan plan.json --confirm-cooperative-writers

# npm
npx artifact-graph restructure inspect --root . --input request.json --format json
npx artifact-graph restructure plan --root . --input mapping.json --format json
npx artifact-graph restructure apply --root . --plan plan.json --confirm-cooperative-writers
```

`plan` reports blockers, unresolved items, candidate issues and out-of-scope reference sites;
`applicable` is false while any blocker or unresolved item remains, and an inapplicable plan must not
be applied. Keep the plan document in an ordinary directory outside the write set — recovery depends
on that document, not on process state.

> **Adoption limits.** The file-set write capability has `candidate` maturity. The qualified
> environment is Darwin / arm64 / APFS only; other platforms are reported as unavailable rather than
> degraded to a non-transactional write. It assumes cooperative writers and does not prove that
> premise: `apply` and `prune-recovery` require `--confirm-cooperative-writers`, and refuse to write
> without it. Recovery requires `recover` with both `--confirm-all-participants-stopped` and
> `--confirm-exclusive-maintenance`. Recovery materials are retained by default; only an explicit
> `prune-recovery` removes them. No cross-platform transactional guarantee is offered. `inspect` and
> `plan` do not write to the project.

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

When only the edge produced by one renumbering should go, name it explicitly instead of sweeping
every orphan:

```bash
pnpm exec artifact-graph version-lock refresh --changed-only --worktree --remove-orphan-edge <edgeId>
```

`--remove-orphan-edge` is repeatable, applies only to edges that are still orphaned, and is mutually
exclusive with `--remove-orphans` — passing both is rejected. Naming an edge that is still live is
rejected without deleting anything, and pre-existing orphans in the same lock file are left intact.

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
