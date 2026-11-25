# artifact-graph

[English](README.md) · **在线文档站（GitHub Pages）：** [https://ifoohoo.github.io/artifact-chain-assistant-site/](https://ifoohoo.github.io/artifact-chain-assistant-site/)

`artifact-graph` 是面向 AI 编程与协作研发流程的 Git 原生制品图扫描、关系校验与版本锁工具。它把项目中的需求、场景、设计说明、源码、测试和追溯版本锁连接成一张可验证的图，让智能体或开发者在宣布“功能已完成”之前，先用确定性的本地命令核清上下文、影响面与证据边界。

默认解析 Markdown 制品；项目注册自定义类型时也可声明 `format: json` 与 `idField`，将原始 JSON 文件直接纳入同一张图。对于尚未确定归属仓库的早期点子，还提供零配置的跨项目轻量收件箱（`artifact-graph inbox`）。

<!-- release-skill:capability:external-write-boundary -->
> **外部写入边界：** 安装 `artifact-graph` 不会修改目标项目，也不会向任何远端系统写入数据。`--help`、`doctor`、`validate`、`query`、`impact`、`coverage`、`inbox list`、`inbox match`、`version-lock audit`、不带 `--conclusion-output` 的 `check-professional` 以及 `read-proof` 均为纯只读命令。`--conclusion-output` 仅在显式指定的绝对路径排他创建一份新证明文件，绝不覆盖已有文件。只有显式执行 `init`、`scan`、`inbox add`、`version-lock refresh`、`hooks install-git` 或 `hooks uninstall` 等命令时，才会在本地受控路径写入文件。

<!-- release-skill:capability:safe-first-command -->
> **安全的第一步：** 在授权初始化配置、刷新版本锁或安装 Git Hook 之前，先运行以下只读命令确认 CLI 与运行环境正常。

```bash
pnpm exec artifact-graph --help
pnpm exec artifact-graph doctor --root .
```

若任一只读命令失败，请先确认当前 Node.js 版本满足 `>=22.22.2 <23`，再按 [INSTALL.md](INSTALL.md) 的说明重新安装。在 CLI 能够正常解析且 `doctor` 给出明确诊断前，不要执行任何写入命令。

## 安装

```bash
pnpm add -D artifact-graph
```

或直接从 GitHub 安装：

```bash
npm install --save-dev github:ifoohoo/artifact-graph
```

运行环境要求 Node.js `>=22.22.2 <23`。使用 pnpm 10+ 时需放行原生模块构建白名单（`better-sqlite3`），配置方式见 [INSTALL.md](INSTALL.md)。

## 快速开始

### 场景 A：想法尚未选定具体仓库（跨项目轻量收件箱）

当只有一个原始想法、还不打算初始化完整项目配置时，可直接写入工作区收件箱并自动检索近似历史想法、用户故事或需求条目：

```bash
npx artifact-graph inbox add "支持审阅结果绑定目标文件哈希，防止串单" --context "跨会话异步复审"
npx artifact-graph inbox list
npx artifact-graph inbox match "审阅结果防串单"
```

该流程默认写入 `.artifact-inbox/ideas.jsonl`（状态固定为 `captured`），不会生成 `artifact-graph.config.yaml`，也不会触发任何脚手架或开发动作。

### 场景 B：在目标仓库建立制品链与版本锁

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

> 首次建立版本锁基线请使用 `version-lock refresh --all`。`--changed-only --staged` 仅用于已有基线项目的 `pre-commit` 增量刷新，不用于首次建锁。

## 核心工作流与边界

### 1. 代码追溯与扫描范围

源码通过独占一行的 `// @feature A1` 注释声明实现或验证关系，技能 Markdown 使用 `<!-- @feature A1 -->`（填写制品 ID `A1`，不要写成命令行选择器 `feature:A1`；暂不支持 Python `#` 注释）。承载追溯注释的源码、测试与技能文件必须纳入 `types.test.paths`，默认配置不会全盘扫描所有 `src/` 或 `skills/` 目录。

> `version-lock audit --strict-missing-lock` 检查的是**已被扫描发现的关系**是否建锁且哈希新鲜。未加注释的源文件或未声明关系的制品不会触发缺锁报错，因此即使锁条目为零也可能通过审计——它不能单独证明“所有发布文件均已覆盖”。完整语法、边分类与多项目隔离规则见 [代码追溯接入说明](INSTALL.md#code-traceability-and-coverage-boundaries)。

### 2. 公开项目类型清单与采用权归属

自 `0.15.0` 起，包内通过子路径 `artifact-graph/project-artifact-profiles` 导出静态推荐清单（配套 Schema 为 `artifact-graph/schemas/project-artifact-profiles.schema.json`，`schema_version` 为 `"1.0"`）：

```js
const profiles = require('artifact-graph/project-artifact-profiles');
```

- `common_minimum` 包含三项跨项目最小类型：`feature`、`scenario`、`design`。
- `project_standards` 当前批准的独立标准为 `skill-plugin`，在 `common_minimum` 基础上增加 `agent_skill` 与 `e2e_test`。
- `lifecycle` 与 `conditional` 仅描述不同采用阶段和能力条件下的候选类型；读取这份 JSON 不会自动启用任何类型。

**采用权始终归目标项目的 `artifact-graph.config.yaml` 所有**：项目已声明的类型与方法绑定（含未出现在公开清单中的自定义类型）均原样保留。`loadConfig` 与 `artifact-graph doctor` 会校验配置结构：未知顶层键产生可定位的兼容性警告而不粗暴拒绝；`allowedEdges` 为空数组 `[]`、`null` 或空值时沿用内置 `DEFAULT_SCHEMA.allowedEdges`，一旦写成非空数组即作为严格的 `from/to/kind` 白名单替换默认表（表外关系报 `DISALLOWED_EDGE`，`forbiddenEdges` 仍优先报 `FORBIDDEN_EDGE`）。缺失配置文件的目录保持“未接入”状态。

### 3. 自定义类型的 JSON 原文件接入

内置类型使用专用解析器；项目自定义的制品类型若需直接纳管 JSON 原文件，须同时声明 `format: json` 与根对象主键属性名 `idField`（未声明 `format: json` 时，即使路径匹配 `*.json` 仍按 Markdown 解析）：

- 每个 JSON 文件须包含单个根对象并映射为一个图节点，`idField` 按完整属性名读取（不按点号拆分），读取后再按 `idPatterns` 校验。JSON 语法错误、根值为数组/标量或编号字段缺失/为空字符串时输出诊断且不生成节点；编号不符合 `idPatterns` 时保留节点并附带诊断。
- 关系字段仍在 `relationSemantics` 中配置。在 JSON 原文件里，`fields` 中的点号表示逐层属性访问（如 `capability.capabilityRef` 依次读取根对象的 `capability` 与子对象的 `capabilityRef`；不支持 `*` 通配或 JSONPath 表达式）。属性缺失时不创建可选边；中间层级非对象或末端值既非非空字符串也非非空字符串数组时，诊断会指出具体文件与字段路径。

示例配置（注意非空 `allowedEdges` 会替换默认表，须显式列出自定义边）：

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
    label: 方法适用的能力
    targetTypes: [capability]
    fields: [capability.capabilityRef]
  releases:
    label: 发行记录对应的方法
    targetTypes: [method]
    fields: [methods]
allowedEdges:
  - { from: method, to: capability, kind: applies }
  - { from: skill_release, to: method, kind: releases }
```

对应的方法文件与发行记录文件示例：

```json
{
  "methodId": "METH-1",
  "capability": { "capabilityRef": "CAP-1" }
}
```

```json
{
  "releaseId": "REL-1",
  "methods": ["METH-1", "method:METH-2"]
}
```

`scanArtifacts`、`validate` 与 `version-lock` 消费同一张图并按原文件字节计算哈希。文件进入图只代表声明关系成立，领域评审与真实发布仍须提供各自的执行证据。

### 4. 日常高频命令速查

- **配置与链接校验**：`artifact-graph init` 生成配置骨架；`artifact-graph validate` 检查断链与非法边。
- **可复用的图专业证明**：`artifact-graph check-professional --root . --format json` 只读组合 `validate`、`version-lock audit` 与 `coverage`，不写图缓存；传入 `--conclusion-output <绝对路径>` 时排他生成证明文件。用 `artifact-graph read-proof --proof-root <root> --proof <相对路径> --format json` 核验本族证明（`GRAPH_CLEAR` 且范围完整时为 `pass`，其余问题或未完成为 `not_pass`，文件缺失/损坏或非本族证明为 `unavailable`；该证明与 Review Result Protocol 分属两套独立契约）。
- **审阅结果校验与防串单**：`artifact-graph validate-review-result --file <path>` 校验协议结构与身份独立性；加上 `--verify-target-digests`（可配合 `--targets <p1,p2>` 或内联摘要映射）可强制比对目标文件当前字节的 SHA-256 摘要，拦截“审阅 JSON 语法合法，但目标文件已改动或串到了其他条目”的风险。
- **上下文组装与影响面**：`artifact-graph context`、`artifact-graph packet` 生成受控实现上下文；配置 `statusViews` 后可加 `--view current|planned|history|all` 按时间视图筛选。`artifact-graph impact --worktree` 只读评估工作树改动的波及范围；`artifact-graph coverage` 报告图扫描与覆盖边界（不推断测试通过或发布完成）。
- **版本锁维护**：`artifact-graph version-lock refresh` 与 `audit` 同时守护实现/验证边（`locks`）和制品间关系（`artifactRelations`），并对带追溯注释的代码测试文件校验其包含真实测试用例或断言，拦截纯注释空壳。旧版 1.0 锁文件缺少 `artifactRelations` 时视为空列表，跑一次 `refresh --all` 即可补齐完整基线。
- **实现表面与双向一致性对齐**：项目配置 `implementationSurface`（`entryGlobs`、`excludeGlobs`、`requireCodeLocks`）后，可用 `artifact-graph alignment-audit [--root .] [--strict] [--format json|markdown]` 审计单仓入口认领，以及跨仓 `artifacts/portfolio-allocations.json` 与 `artifacts/upstream-bindings.json` 的对称绑定、血缘子集新鲜锁与只读收据（严格区分静态双锁齐备 `traceability_locked` 与具备真实测试执行通过收据的 `verified_satisfied`，并支持已吸纳条目声明 `allocationStatus: "absorbed"` 与 `absorbedBy` 承接仓）；显式运行 `artifact-graph alignment-lock` 同步互锁摘要。
- **Git Hook 安装与幂等卸载**：`artifact-graph hooks install-git --hook all` 安装可选钩子；停用时执行 `artifact-graph hooks uninstall --hook all`，仅幂等移除受管片段并完整保留用户自定义的 Hook 命令与文件权限。

### 5. 制品重组（拆分、跨文件移动与重编号）

`artifact-graph restructure` 将显式映射编译为可复核的候选计划并按文件集合应用，支持三类确定性变换：`record-split`（物理记录拆分）、`identity-split`（编号身份一拆多）与 `move-renumber`（跨文件迁移与重编号）。能力边界划分与验收条件归属由上层技能或作者决定，CLI 只编译完整映射并执行文件切换：

```bash
pnpm exec artifact-graph restructure inspect --root . --input request.json --format json
pnpm exec artifact-graph restructure plan --root . --input mapping.json --format json
pnpm exec artifact-graph restructure apply --root . --plan plan.json --confirm-cooperative-writers
```

只要 `plan` 中仍存在阻断项或未决项，`applicable` 即为 `false`，禁止进入 `apply`。请将计划文件保存在候选写集之外的普通目录——中断恢复只依赖该计划文件，不依赖内存进程状态。

> **采用边界：** 文件集合写入能力当前成熟度为 `candidate`，仅在 `Darwin / arm64 / APFS` 环境下开放真实写入；其他平台仍可使用只读的 `inspect` 与 `plan`，调用 `apply` 会以 `APPLY_ENVIRONMENT_UNSUPPORTED` 明确拒绝，绝不静默降级为无事务裸写。`apply` 与 `prune-recovery` 要求操作者显式传入 `--confirm-cooperative-writers`；中断恢复须执行 `recover` 并同时传入 `--confirm-all-participants-stopped` 与 `--confirm-exclusive-maintenance`。恢复材料默认保留，仅在显式执行 `prune-recovery` 时清理。

### 6. 删除或拆分制品后的孤立锁清理

删除、合并或重编号制品后，旧边对应的版本锁会变成孤立锁（orphan lock）。`version-lock refresh` 默认保留孤立锁，确保清理动作始终可见、可审阅：

```bash
# 批量清理全部孤立锁
pnpm exec artifact-graph version-lock refresh --all --remove-orphans --format markdown
git diff artifacts/traceability-version-lock.json
git add artifacts/traceability-version-lock.json

# 仅精准移除本次改号产生的单条孤立边（与 --remove-orphans 互斥，可重复传参）
pnpm exec artifact-graph version-lock refresh --changed-only --worktree --remove-orphan-edge <edgeId>
```

若点名的 `edgeId` 当前仍是有效活边，命令会拒绝删除并保留原状。`version-lock audit` 会将结构性孤立锁与过期哈希列为阻断项；测试文件脱离已配置运行器激活范围（runner liveness）则仅报警告。

## Review Result Protocol（审阅结果协议 v1.0）

本包导出 `schemas/review-result.schema.json`、TypeScript 类型定义及 `validateReviewResult(input, options?)` 语义校验函数，统一约束审阅（`review`）、修复（`repair`）、批次度量、问题分级与阻断裁决：

- 拒绝未知顶层字段；`attempt` 限制在 `1..3`；验收通过（`SUCCEEDED` + `PASS` / `PASS_WITH_RESIDUAL_MINOR`）必须记录 `producer` 身份，且不允许残留未关闭的 `block` 级问题。
- 独立复审可通过 `acceptance.reviewer` 与 `acceptance.source_result` 关联修复来源；语义校验器按 `executor + name` 比对稳定身份（`skill` 仅作附加元数据），拒绝修复执行者自我验收。
- 当向 `validateReviewResult` 传入 `{ expectedTargetDigests }`（或 CLI 传入 `--verify-target-digests`）时，校验器会进一步核对 `review.target_digests` / `evidence` 中的目标文件 SHA-256 摘要及 `review.files` 范围，防止旧快照混入或跨条目串单。

## 文档导航

- [在线文档站（GitHub Pages）](https://ifoohoo.github.io/artifact-chain-assistant-site/) — 产品概览、十分钟上手、图谱与四态对齐、工作流协同、命令行参考与排障指南
- [INSTALL.md](INSTALL.md) — 安装要求、pnpm 10+ 原生模块白名单与代码追溯边界说明
- [CHANGELOG.md](CHANGELOG.md) — 版本变更记录
- [CONTRIBUTING.md](CONTRIBUTING.md) — 参与贡献指南
- [SECURITY.md](SECURITY.md) — 安全漏洞报告策略

## 配套项目

如需在 Codex、Claude Code、Kimi Code 或 Qoder 中使用引导式技能完成制品链初始化、需求池双维管理、现状分诊与规格编写，请配合安装 [`artifact-chain-assistant`](https://github.com/ifoohoo/artifact-chain-assistant)。

## 开源许可

Apache-2.0。详见 [LICENSE](LICENSE)。
