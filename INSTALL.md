# artifact-graph Installation

## Prerequisites

- Node.js `>=22.0.0`.

## Install

### From npm

```bash
pnpm add -D artifact-graph
```

Or with npm:

```bash
npm install --save-dev artifact-graph
```

### From GitHub

```bash
npm install --save-dev github:ifoohoo/artifact-graph
```

Or with pnpm:

```bash
pnpm add -D github:ifoohoo/artifact-graph
```

After installation, verify the CLI is available:

```bash
# pnpm
pnpm exec artifact-graph --help

# npm
npx artifact-graph --help
```

### pnpm Native Build Allowlist

`artifact-graph` depends on `better-sqlite3`, which requires a native build. pnpm blocks postinstall
scripts by default; you must explicitly allow the build. The configuration key depends on your pnpm
version:

**pnpm 10.26+** — add `allowBuilds` to `pnpm-workspace.yaml` in your project root:

```yaml
# pnpm-workspace.yaml (pnpm 10.26+)
allowBuilds:
  better-sqlite3: true
```

**pnpm 10.0–10.25** — add `onlyBuiltDependencies` to your project `package.json`:

```jsonc
// package.json (pnpm 10.0–10.25)
{
  "pnpm": {
    "onlyBuiltDependencies": ["better-sqlite3"]
  }
}
```

Without the correct entry for your pnpm version, `pnpm install` may skip the native build and
`artifact-graph` will fail at runtime with a missing binding error.

## Quick Start

After installing as a dev dependency, use your package manager's exec to invoke the CLI. From your
project root:

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

## Code Traceability And Coverage Boundaries

Version locks track declared relationships and changes to their file contents. A successful
`version-lock audit --strict-missing-lock` does **not** prove that every release file has an
artifact link or that every artifact has an implementation. Read this section before using
the audit as a release acceptance check.

### Declare a source-to-artifact link

Use a standalone line comment in source files whose language accepts `//`:

```ts
// @feature A1
// @scenario S-01 @feature A1
// @decision D-TOOL-01
```

The syntax is `@<registered-type-or-alias> <artifact-ID>`. The tag selects the type; the value
is the ID, not `feature:A1`, a Markdown link, or a file path. Separate multiple IDs with spaces
or commas, or repeat the tag. Each ID must match the configured type's `idPatterns`. Custom
types and their explicit aliases are supported when registered in the project configuration.
Keep explanatory prose on a separate line. Split mappings that combine multiple scenarios
and multiple features into separate lines so their correspondence is unambiguous.

For a Markdown implementation such as a skill, use a standalone, single-line HTML comment:

```markdown
<!-- @feature A1 -->
```

Place the comment outside fenced examples so it declares the file's own responsibility.
The scanner is a text parser, not a language compiler. It recognizes standalone `//` and
single-line `<!-- ... -->` forms. It does not support native Python/shell `#` comments, SQL
`--` comments, or `/* ... */`/JSDoc traceability blocks. Trailing `//` comments after code are
invalid. Do not put invalid-language `//` comments into Python files to work around this limit.
JavaScript, TypeScript and Java are examples of languages that can use the `//` form;
this is not a promise of complete language-specific syntax parsing.

### Configure the files to scan

Source annotations are parsed through `types.test.paths`, including implementation files.
The historical name `test` does not mean that every matching file becomes a test node.
Merge paths appropriate to the project into its existing `artifact-graph.config.yaml`:

```yaml
types:
  test:
    paths:
      - "src/**/*.ts"
      - "src/**/*.tsx"
      - "tests/**/*.ts"
      - "skills/**/*.md"
idPatterns:
  test: '^.+\.(ts|tsx|md)$'
```

These are example source paths, not a replacement for the project's artifact type definitions.
The built-in default only matches `heimdall/packages/**/*.test.ts`; ordinary `src/` and
`skills/` trees require configuration. Directory walking skips `node_modules`, `dist`, `.git`
and `.artifact-graph`. Scanning does not derive scope from a release manifest or `.gitignore`.
If types overlap, the most specific path definition claims the file; check the
`ARTIFACT_PATH_OVERLAP` diagnostic rather than assuming both parsers run.

Files named `*.test.*`, `*.spec.*`, `*Test.java` or `*Tests.java`, and files under `test/`,
`tests/` or `__tests__/`, are classified as tests and create `verifies` locks. Other annotated
files create implementation nodes and `implements` locks. A plain file with no recognized
annotation produces no source node. A skill Markdown file claimed by a custom artifact type
is an artifact node, not automatically a code implementation; confirm `sourceKind` in the
version index before choosing an implementation-lock policy.

### Establish and check the locks

After declaring real links and verifying their targets, run:

```bash
pnpm exec artifact-graph version-index --root . --format json
pnpm exec artifact-graph validate --root . --format json
pnpm exec artifact-graph version-lock refresh --root . --all --format markdown
pnpm exec artifact-graph version-lock audit --root . --strict-missing-lock --format json
```

Inspect the index for the intended source nodes and edges. `validate` checks annotation/link
errors; `refresh --all` records current hashes for discovered relationships; `audit` checks
those locks. These commands do not execute the source or prove that an implementation meets
its requirements. Use a separate, existing project test gate for behavioral verification.
Do not automatically refresh immediately before the acceptance audit merely to clear stale
hashes: first review the changes and run the relevant tests.

To lock just one declared relationship, use:

```bash
pnpm exec artifact-graph version-lock update --root . --target feature:A1 --source skills/example/SKILL.md
```

`update` requires both `--target` and `--source`; there is no `update --all` operation.
`bootstrap` can establish an initial baseline, but `refresh --all` is the recommended
incremental entry. Do not use `bootstrap --force` in routine maintenance or hooks.

### What missing_lock counts

`missing_lock` is reported per discovered, lockable relationship without a corresponding lock:

- Source-to-artifact `implements`/`verifies` relationships are checked against `locks`.
- Artifact-to-artifact relationships are checked against `artifactRelations` too.
- With runners configured, tests in discovery scope but inactive in every matching runner
  are excluded from automatic lock creation and missing-lock counting. Unscoped tests remain
  lockable. Without runner configuration, the legacy fallback treats non-E2E test paths as
  active; E2E paths need a recognized `// @e2e_test` or `// @tc` annotation to be active.
  These liveness rules are not a general release-file exemption mechanism.

Missing locks are warnings by default; `--strict-missing-lock` makes them blocking without
changing the population being checked. Do not combine a required acceptance check with
`--warning-only`, which suppresses its nonzero exit status.

An artifact with no relationships does not produce `missing_lock` just because it has no
implementation. An artifact without incoming implementation links can still have outgoing
artifact relationships that require relation locks. `totalLocks: 0` only counts implementation
and verification locks; inspect `totalArtifactRelationLocks` separately. With no lockable
relationships and no existing locks, even the strict audit can pass with zero locks.

### Release coverage and exemptions

The `coverage` command reports graph health, scan mapping, declared behavior references, and an
optional caller-supplied release-file mapping. It is a boundary report, not a release-payload gate:

```bash
pnpm exec artifact-graph coverage --root . --format json
pnpm exec artifact-graph coverage --root . --release-input release-files.txt --format json
```

Behavior verification is reported as `not-evaluated`; implementation, verification, and evidence
edges remain raw declaration references. With a release input list, mapped and unmapped paths are
reported but release status remains `unknown` and evidence remains empty. Without a list, release
status is `not-requested`. The command never turns a file list into proof that a version was
published. Graph health accounts for validation and lock issues. Scan coverage reports both files
inside configured scan paths and files actually mapped to nodes or edge sources; changed-path
classification is marked unavailable outside a Git worktree. There is no general artifact/file exemption marker for `missing_lock`; E2E coverage
waivers concern a different check and must not be used as implementation-lock exemptions.

For a policy requiring every release code file to link upward and every required artifact to
have an implementation or approved exemption, keep two separate checks:

1. Use the existing release process's authoritative payload list to compare expected source
   files with the version index and implementation locks. Compare the required artifact set
   with incoming implementation locks and the project's explicitly approved exceptions.
2. Use `validate` and strict version-lock audit for declared links and their freshness, plus
   the project's behavioral tests. These checks cannot replace the coverage comparison.

The acceptance decision remains a project/release-policy responsibility. `coverage` can compare
the caller's existing inventory with graph mapping, but does not own that inventory or approve its
exceptions. Reuse existing release inventory and approval records; do not maintain a second
payload manifest or invent an artifact-graph exemption field. If no authoritative execution or
release result exists, keep the corresponding fact unknown rather than accepting the mapping or
lock audit as proof.

### Time views and read-only impact

Projects may map their own status words into `current`, `planned`, and `history` with
`statusViews`. Explicit views are available on graph queries and context assembly; omitting
`--view` preserves the earlier unfiltered behavior:

```yaml
statusViews:
  active: current
  accepted: current
  planned: planned
  open: planned
  done: history
  deprecated: history
```

```bash
pnpm exec artifact-graph query --from feature:A1 --view current --format json
pnpm exec artifact-graph context --target feature:A1 --view current --format json
pnpm exec artifact-graph packet --target feature:A1 --view current --format json
```

When a project configures `supersedes`, a full replacement moves its target to history only while
the replacing artifact maps to `current`. A planned or uncategorized replacement cannot invalidate
the current baseline. A partial replacement keeps the old artifact in its own view and reports the
replaced sections when the replacing artifact is current.

Use `impact` when selecting work or tests without refreshing locks:

```bash
pnpm exec artifact-graph impact --root . --worktree --format json
pnpm exec artifact-graph impact --root . --staged --format json
pnpm exec artifact-graph impact --root . --base main --format json
pnpm exec artifact-graph impact --root . --paths src/a.ts,src/b.ts --format json
```

The report separates directly mapped nodes, related nodes and edges, graph-control files, files in
configured scan scope that did not resolve to a node, and paths outside scan scope. Generating the
report does not refresh locks or write graph state.

Exclude tooling such as non-shipped `scripts/check-*.py` from the release coverage denominator
using the release inventory. Omitting those files from annotation scan paths is a scan-scope
decision, not a machine-validated exemption. Shipped compatibility forwarding shells still
belong in the payload comparison: link them to the artifact describing compatibility behavior,
or record an exception in the project's existing approval process. An ADR that is intentionally
documentation-only need not be assigned a fabricated implementation link.

### Store locks per project

Commit `artifacts/traceability-version-lock.json` with the project and review its diff; do not
gitignore it. The lock stores project-relative paths and content hashes so another checkout
can audit the same baseline. Generated caches such as `.artifact-graph/` are separate.
Use each project's own `--root` and lock file even when projects share one CLI installation.
If a project uses `--lock-path`, pass the same project-relative path to every lock command and
its hooks/CI; the flag is an invocation option, not a persistent setting. Keep one authoritative
lock per effective project graph, rather than sharing a lock across unrelated roots.

## Cleaning Up Orphan Locks

After deleting or splitting an artifact (or removing a traceability edge), the affected version
locks become orphans. Refresh retains them by default; clean them up explicitly and review the
result before staging:

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

`version-lock audit` reports structural orphan locks and stale hashes as blocking issues with
remediation steps; runner liveness findings are non-blocking warnings.

## Smoke Test

Run these commands to confirm the installation is working:

```bash
# pnpm
pnpm exec artifact-graph --help
pnpm exec artifact-graph doctor --format markdown
pnpm exec artifact-graph validate --root . --warning-only

# npm
npx artifact-graph --help
npx artifact-graph doctor --format markdown
npx artifact-graph validate --root . --warning-only
```

If `artifact-graph doctor` cannot find the CLI or config, check that:

1. `artifact-graph` is in `./node_modules/.bin/` (run `pnpm exec artifact-graph --help` or
   `npx artifact-graph --help` to verify).
2. Your project has an `artifact-graph.config.yaml` (run `pnpm exec artifact-graph init --root .` or
   `npx artifact-graph init --root .` to create one).

## Universal Baseline Policy

Starting with 0.5.0, `artifact-graph context` and `artifact-graph packet` inject 19 always-present
baseline files (AGENTS.md, CLAUDE.md, artifact-chain-spec, blueprints, contracts, domain artifacts,
verification files, etc.) as required context by default. When any of these files is missing or
unreadable, the context manifest reports them in `missingDetails` and the command exits non-zero.

### Default behavior

```yaml
# artifact-graph.config.yaml
# context.universal_baseline defaults to true — no explicit entry needed
```

With the default, all baseline files are verified against the project root. If a file is missing,
is a directory, or is unreadable, it appears in the structured `missingDetails` with kind
`missing-baseline`.

### Explicit opt-out for lightweight projects

If your project does not contain all 19 baseline files (e.g., a partial migration or a standalone
library), explicitly disable baseline injection:

```yaml
# artifact-graph.config.yaml
context:
  universal_baseline: false
```

With `false`, `resolveArtifactContext` skips baseline injection entirely. No `baseline` category
appears in the context manifest, and the manifest writes `baselinePolicy: false` so that
packet validation (`validatePacket`) correctly allows `requiredBaseline.total=0`.

### Config validation

`loadConfig` rejects non-boolean values for `context.universal_baseline`:

| Value | Result |
|-------|--------|
| `true` | Baseline enabled |
| `false` | Baseline disabled |
| `undefined` | Defaults to `true` |
| `0`, `1` | **Error**: `Invalid context.universal_baseline` |
| `""`, `"false"`, `"true"` | **Error**: `Invalid context.universal_baseline` |

### Migration impact

- **Existing projects with all baseline files present**: no change in behavior. The default
  `true` policy was already implicit in 0.4.x context resolution.
- **Projects missing baseline files**: add `context.universal_baseline: false` to suppress
  baseline verification, or create the missing files. Without this, `context` and `packet`
  commands will exit non-zero with structured missing evidence.
- **Packet validation**: `validatePacket` (PKT-004) now requires an explicit `baselinePolicy`
  field to allow `requiredBaseline.total=0`. Packets without `baselinePolicy` and with
  `total=0, missing=[]` are rejected — this prevents silent opt-out inference.

## Related Project

Use [`artifact-chain-assistant`](https://github.com/ifoohoo/artifact-chain-assistant) for Codex and
Claude Code skills that guide artifact-chain intake, setup, and maintenance.
