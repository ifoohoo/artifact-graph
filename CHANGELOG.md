# Changelog

## 0.9.4

### Changed

- Version synchronized with `artifact-chain-assistant@0.9.4` as a suite lockstep release. This
  release does not change `artifact-graph` runtime behavior.

## 0.9.3

### Fixed

- Scenario titles and explicit relation fields inside Markdown fenced code blocks
  (```` ``` ```` or `~~~`) are no longer scanned: examples such as
  `**关联决策**: ADR-0001` shown inside a fence produce neither scenario nodes nor graph
  edges. A fence closes only on the same marker with a length at least as long as the
  opening fence.
- Explicit relation fields indented by four or more spaces or a tab are now treated as
  indented code and no longer produce edges. Outside code blocks, a real relation field
  may be indented by at most 0—3 plain spaces. Behavior tightening: adopters who relied
  on deeply indented relation fields must move them to line start (0—3 spaces).
- The public `validateScenarioPrdLinks` function once again returns the complete
  diagnostics (including `FORMAT_ERROR` for invalid feature references) when called
  directly on scan results, restoring the public API behavior regressed in 0.9.2.
- `validateGraph` still uses scan diagnostics as the single source for scanned invalid
  relations, so the same invalid feature reference is reported exactly once.

## 0.9.2

### Fixed

- Scenario explicit relation field recognition is tightened to whole-line field syntax matching:
  plain prose mentions of `关联功能` / `关联决策` / `关联实体` no longer produce edges or
  spurious `FORMAT_ERROR` diagnostics.
- An invalid explicit relation reference is no longer reported twice across the scan and
  validation stages: `validateGraph` now returns exactly one `FORMAT_ERROR` carrying the file,
  line number, node, invalid value, and the expected rule.
- Behavior tightening: Markdown list-item forms such as `- **关联决策**: ADR-0001` are no longer
  recognized as explicit relation fields (0.9.1 misidentified them via substring matching);
  standard line-start field syntax is unaffected.

## 0.9.1

### Fixed

- Scenario explicit relation fields (`关联决策`, `关联功能`, `关联实体`) and design-document
  frontmatter `related_decisions` now validate each reference against the project's configured
  `idPatterns`, so custom identifier schemes such as `ADR-0002` or `FEAT-12` produce graph edges.
  Invalid identifiers emit diagnostics with file and line numbers instead of being silently
  dropped, while explicit empty markers such as `无` produce neither edges nor diagnostics.
- Design documents now read `related_decisions` as a structured frontmatter field (edge source
  `frontmatter`), scan only the body for prose references, and never emit duplicate edges for
  the same relation.
- `query --from` and `render --from` now resolve a bare identifier to its unique graph node
  regardless of the configured `idPatterns`, and report an ambiguity diagnostic asking for a
  full `type:id` when multiple artifact types share the same bare identifier.

## 0.9.0

### Changed

- Version synchronized with `artifact-chain-assistant@0.9.0` (suite lockstep release). No runtime
  behavior changes.

## 0.8.5

### Fixed

- E2E trace validation now accepts only real standalone line comments. Annotation-shaped text in
  source strings and template literals is ignored, eliminating false `E2E-TRACE-002` findings from
  test fixtures.
- Staged version-lock refresh now blocks only for unstaged files matched by the project's configured
  artifact paths (plus the graph config and lock file). Unrelated Markdown or source files no longer
  prevent a valid split commit, while genuine graph-relevant divergence remains fail-closed.

## 0.8.4

### Changed

- Version-lock audit and refresh Markdown now explains findings in Chinese, labels each item as a
  blocking error or non-blocking warning, and includes concrete review, refresh, cleanup, and staging
  commands.
- `missing_lock` remains non-blocking by default and is promoted only by
  `--strict-missing-lock`; runner liveness findings are warnings and no longer fail commits on their
  own. Markdown summaries and CLI exit codes now share the same policy decision.
- `version-lock refresh --all --remove-orphans` is documented in CLI help and remediation output as
  the safe cleanup path after deleting, renaming, or splitting artifacts. Structural orphan locks
  remain blocking until the user explicitly reviews and removes them.

### Fixed

- Pre-commit guidance now distinguishes a successful refresh that changed the lock file from a
  validation failure, and gives the exact review, stage, and retry steps.

## 0.8.3

### Fixed

- Restored the complete npm runtime payload: `dist/cli.js`, ESM/CJS modules, declarations, and
  `contracts/e2e-test/schema.json` are now present in the published tarball. Versions 0.8.0 through
  0.8.2 omitted these generated entry points and cannot provide the declared CLI or library exports.

### Changed

- The release build now rebuilds `dist/`, verifies every `package.json.files` path against the
  release-skill public snapshot, requires every manifest entry point, installs the packed consumer,
  and configures a post-publish CLI smoke check.

## 0.8.2

### Changed

- **Version bump for dual-repo synchronization with `artifact-chain-assistant@0.8.2`**. No runtime
  code changes in this package; published solely to keep the dual-release version pair in lockstep.

## 0.8.1

### Changed

- **Version bump for dual-repo synchronization with `artifact-chain-assistant@0.8.1`**. Includes README
  documentation polish, three new entry skills (`help`, `setup`, `quickstart`), external marketplace
  (`ifoohoo/artifact-skill-set`) installation support, and release gate fixes.

## 0.8.0

### Added

- **Artifact relation version locks**: `version-lock refresh` and `bootstrap` now generate
  `artifactRelations` entries for artifact-to-artifact edges (for example, skill→skill_family and
  feature→scenario). The `audit` command checks for missing relation locks, content hash mismatches,
  and path drift.

### Changed

- **Strict local artifact-relation refresh**: `version-lock refresh --changed-only` updates or adds
  only relation locks whose endpoints are reached by the current worktree or staged change set. It
  no longer fills unrelated missing relation locks while processing a local change; use
  `version-lock refresh --all` only when intentionally establishing a complete relation baseline.
- **Deterministic, reproducible relation locks**: relation lock output uses deterministic sorting and
  carries no timestamps, so identical artifact content produces byte-identical lock files across
  environments and runs.
- **Old 1.0 lock read boundary**: lock files written before artifact relations existed (no
  `artifactRelations` key) are read as an empty relation list instead of failing; run
  `version-lock refresh --all` once to establish the complete relation baseline.
- **Non-software domain isolation via `baselinePolicy: false`**: when a project explicitly disables
  the universal baseline (`baselinePolicy: false` in the context/packet manifest), packet assembly
  and validation no longer inject the software-development defaults — baseline constraints, the risk
  checklist, and blueprint non-goals are required to be empty (PKT-005 / PKT-010), so non-software
  skill-family domains are not forced through software-specific packet rules.

## 0.7.0

### Added

- **Contract kernel**: new deterministic contract kernel providing contract identity, a contract
  registry, canonical IR normalization, revision digests, and policy compatibility checks. Exposed
  through the new `artifact-graph contract` CLI subcommand group and public library exports.

### Changed

- **Version-lock changed-only collects untracked paths**: `version-lock refresh --changed-only`
  (worktree and staged modes) now also collects untracked files, so newly added artifacts can no
  longer silently bypass lock refresh.
- **Organization migration to `ifoohoo`**: the public repository transferred to the `ifoohoo`
  GitHub organization (`ifoohoo/artifact-graph`, name unchanged). Copyright is now held by
  广州市风荷科技有限公司 (Guangzhou Fenghe Technology Co., Ltd.) together with the project
  contributors; the NOTICE file states that the organization transfer is an administrative hosting
  change, not a copyright assignment. Package metadata, installation docs, and repository
  references were updated accordingly; the npm package name `artifact-graph` is unchanged.

## 0.6.1

### Fixed

- **CHANGELOG documentation corrections**: correct two 0.6.0 CHANGELOG descriptions to match
  actual implementation:
  - `E2E-UNIT-TEST-NOT-E2E`: the trigger condition is based on `e2e.runners[].kind` and runner
    root/include/exclude/testIgnore acceptance, not `.test.ts` / `.spec.ts` file suffix.
  - `E2E-TRACE-004`: when Markdown explicitly declares `chain_type: desktop_chain`, conflicts with
    source-level `mock_playwright` annotations are entirely suppressed (not downgraded to info).

## 0.6.0

### Added

- **E2E coverage proof mechanism**: `validate --include e2e-coverage` now outputs executable_ref
  coverage statistics (total TCs, with executable_ref, rate), status breakdown, chain_type breakdown,
  uncovered scenarios, and uncovered features. JSON and human-readable formats supported.
  - Configurable thresholds via `artifact-graph.config.yaml` `e2e` section:
    `executable_ref_warning`, `executable_ref_error`, `report_uncovered_scenarios`,
    `report_uncovered_features`, `scenario_waivers`, `feature_waivers`.
  - Threshold violations produce `E2E_COVERAGE_WARNING` / `E2E_COVERAGE_ERROR` findings.

- **TC status lifecycle validation**: TCs with invalid `status` values produce `E2E_INVALID_TC_STATUS`.
  `waived` status requires non-empty `waived_reason` (`E2E_WAIVED_NO_REASON`).

- **chain_type vocabulary validation**: Invalid chain_type produces `E2E_INVALID_CHAIN_TYPE`.
  Deprecated aliases `core_only` → `core_e2e`, `frontend_only` → `mock_playwright` produce
  `E2E_DEPRECATED_CHAIN_TYPE` migration warnings.

- **ac_coverage_rate freetext detection**: Handwritten percentages produce
  `E2E_AC_COVERAGE_RATE_FREETEXT` — this field must be computed, not manually entered.

- **Deterministic checklist rules**:
  - `E2E-UNIT-TEST-NOT-E2E`: executable_ref target is only accepted by unit runner(s)
    (per `e2e.runners[].kind` and runner root/include/exclude/testIgnore acceptance), not by any
    e2e/integration runner.
  - Version-lock liveness: E2E spec files with no active `@e2e_test`/`@tc` annotations produce
    `orphan_lock` liveness warnings.

- **`generate-e2e-registry` command**: Deterministic, idempotent E2E registry generation from
  Markdown test files. `--deterministic` flag sets `generated_at` to epoch for diff checks.
  `--out <path>` writes output to file.

- **E2E-TRACE-004 Markdown authority**: When a TC explicitly declares `chain_type: desktop_chain`
  in its Markdown, the Markdown side is authoritative (per artifact-chain-spec §5.2) — conflicts
  with source-level `mock_playwright` annotations are entirely suppressed (not emitted as warning
  or info).

### Changed

- **`validate --format json`** output is now wrapped in `{ issues, e2eCoverage }` when
  `--include e2e-coverage` is specified. Without `--include e2e-coverage`, output remains a raw
  issues array (backward compatible).

## 0.5.0

### Added

- **Universal baseline policy**: `resolveArtifactContext` now injects 19 always-present baseline
  files (AGENTS.md, CLAUDE.md, artifact-chain-spec, blueprints, contracts, etc.) as required context
  by default. `scanArtifacts` stores a normalized absolute `root` on the graph so that
  `resolveArtifactContext` can fall back to `graph.root` when callers omit `opts.root`.
  - Default: `context.universal_baseline` is `true` (all baseline files injected and verified).
  - Explicit opt-out: set `context.universal_baseline: false` in `artifact-graph.config.yaml` to
    skip baseline injection entirely for lightweight or partial projects.
  - Config validation: `loadConfig` rejects non-boolean values (`0`, `""`, `"false"`, `1`, `"true"`)
    with an explicit error.
  - Fail-closed: when baseline is enabled but no project root is available, all 19 baseline items
    appear in `missingDetails` with kind `missing-baseline`; the manifest writes
    `baselinePolicy: true` so downstream packet validation cannot silently infer opt-out.
  - Readability gate: baseline file checks now verify both `stat.isFile()` and read permission
    (`access(R_OK)`), so unreadable files are reported in `missingDetails` rather than silently
    skipped.
- **`ArtifactGraph.root` field**: `ArtifactGraph` interface gains an optional `root?: string` field
  populated by `scanArtifacts` with the normalized absolute project root. Consumers that construct
  graph literals without this field remain backward-compatible.

### Fixed

- **CLI `--help`/`-h` safety for subcommands**: `artifact-graph hooks install-git --help/-h` and
  `artifact-graph version-lock --help/-h` now exit 0 and print usage without executing command
  side effects. Previously, `hooks install-git --help` would install hooks into the Git repository
  instead of showing help. This prevents accidental hook installation when users pass `--help` to
  verify CLI behavior. Regression tests verify that `--help`, `-h`, and positional `help` tokens
  do not create or modify Git hooks.
- **Review Result input hardening**: reject unknown top-level fields and attempts outside 1–3;
  require producer identity for successful PASS decisions; reject PASS decisions with open block
  findings; and reject repair self-acceptance using stable `executor + name` identity even when
  `skill` metadata differs. This tightens Review Result v1.0 consumption compatibility, including
  acceptance identity rules: migrate legacy top-level fields into protocol sections and run
  `artifact-graph validate-review-result --file <result.json> --format json` before consumption.
- **Canonical E2E code tag**: add `@e2e_test` as the canonical traceability tag; legacy `@tc`
  remains an alias and emits the `E2E-TRACE-007` deprecation warning from generic code-comment scans.
- **Traceability annotation false-positive fix**: reduce false positives in source/test traceability
  comment validation for custom artifact types registered via `artifact-graph.config.yaml`.
- **Pre-commit hook configuration detection**: improve pre-commit hook configuration detection to
  handle non-standard Git hook directory layouts and `core.hooksPath` overrides.

## 0.4.1

### Changed

- Version bump for dual-repo synchronization with `artifact-chain-assistant@0.4.1`. No runtime code changes in this package.

## 0.4.0

### Added

- Add project-neutral Review Result Protocol v1.0 schema, TypeScript types and validateReviewResult validator API.
- Add `validate-review-result --file <path>` with absolute-path support and JSON-path diagnostics.
- Include `schemas/review-result.schema.json` in the published package.

## 0.3.1

### Fixed

- **CLI help contract**: `artifact-graph --help`、`-h` 和 `help` now return exit code 0 and print usage to stdout, making `--help` a reliable install verification gate for public INSTALL instructions.
- **init creates missing directories**: `artifact-graph init --root <path>` now creates intermediate directories if they don't exist, matching the documented usage pattern.

## 0.3.0

### Added

- **Config-driven custom artifact types**: register any Markdown artifact type via `artifact-graph.config.yaml` with `paths`, `idPatterns`, `extraFields`, `target`, `role`, and `aliases`. Generic frontmatter parser handles all registered types without dedicated code.
- **Dynamic `--target` selector**: unified `--target <type>:<id>` for `context`, `packet`, `packet-prompt`, and audit commands; legacy `--feature`, `--scenario`, `--decision`, `--design`, `--e2e-test` flags remain compatible.
- **Extra fields indexing**: declare `extraFields` (string, number, boolean, enum) in config to index specific frontmatter fields for custom types.
- **`packet-prompt-audit --discover` mode**: automatically discovers all targets from artifact config and audits packet prompts without a targets file.
- **`CUSTOM_ARTIFACT_ISOLATED` validation**: warns when config-registered custom types have no traceability edges.
- **Java code traceability**: `isTestFile` classifier supports `*Test.java`/`*Tests.java` convention for Java projects.

### Changed

- Scenario-PRD validation (`--include scenario-prd-links`) is now opt-in via `validateScenarioPrdLinkIndex` instead of unconditionally executed.
- Expand `release-verifier` sub-agent to check CHANGELOG completeness and GitHub Release readiness.
- Document CHANGELOG format standard (Keep a Changelog style with category headings).
- Document version decision rules (semver based on CHANGELOG content).
- Add GitHub Releases creation step to publish workflow.
- Reference release-policy.md for version/CHANGELOG/GitHub Releases procedures in SKILL.md.

## 0.1.4

### Changed

- Publish `artifact-graph` as a standalone public package.
- Add version-lock refresh and audit workflows.
- Add Git hook installation support.
- Add package README, Chinese README, NOTICE, and Apache-2.0 license files.
