# artifact-graph

[English](README.md)

`artifact-graph` 是一个 Git 原生的 Markdown 制品图扫描与校验工具，面向 AI 编程工作流。

它把项目里的需求、场景、设计、源码、测试和 version-lock 元数据连成一张图，
供 AI agent 在宣称实现完成之前，用确定性的本地命令检查上下文和追溯关系。

<!-- release-skill:capability:external-write-boundary -->
> **外部写入边界：** 安装 `artifact-graph` 不会修改项目或写入远端。`--help`、`doctor`、
> `validate`、`query` 和 `audit` 等只读命令不会更改项目文件。只有显式执行 `init`、
> `version-lock refresh` 或 `hooks install-git` 等命令时才会发生本地写入。

<!-- release-skill:capability:safe-first-command -->
> **安全的第一步：** 在授权初始化、刷新版本锁或安装 hook 前，先检查已安装的命令行工具
> 和项目健康状态。

```bash
pnpm exec artifact-graph --help
pnpm exec artifact-graph doctor --root .
```

如果任一只读命令失败，先确认当前 Node.js 满足 `>=22.22.2 <23`，再按照
[INSTALL.md](INSTALL.md) 中的精确说明重新安装；命令行工具能够正常解析且 doctor
给出可处理的诊断前，不要运行写入命令。

## 安装

```bash
pnpm add -D artifact-graph
```

或从 GitHub 安装：

```bash
npm install --save-dev github:ifoohoo/artifact-graph
```

需要 Node.js `>=22.22.2 <23`。pnpm 10+ 需要配置原生构建白名单，详见 [INSTALL.md](INSTALL.md)。

## 快速开始

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

> 首次初始化版本锁使用 `version-lock refresh --all`。`--changed-only --staged` 适用于
> 已有项目的 pre-commit hook，不适用于首次初始化。

## 常见工作流

### 代码与制品追溯：先确认扫描范围

源码可用独占一行的 `// @feature A1` 声明实现关系；技能 Markdown 可用
`<!-- @feature A1 -->`。`A1` 是制品 ID，命令行的 `feature:A1` 写法不能直接放在注释值中。
文件必须纳入 `types.test.paths`；该字段也用于实现文件，默认不会扫描所有 `src/` 或 `skills/`。
Python 的 `#` 注释目前不受支持。

> `version-lock audit --strict-missing-lock` 只检查已发现关系的锁与新鲜度。
> 没有注释的文件、没有关系的制品可能不产生缺锁问题；零锁也可能审计通过。
> 它不能证明“每个发布文件都有制品来源、每个制品都有实现或豁免”。

详细语法、扫描配置、`implements`（实现）与 `verifies`（验证）的分类、豁免边界及多项目隔离，
见 [代码追溯接入说明](INSTALL.md#code-traceability-and-coverage-boundaries)。先确认索引中确有预期节点和关系，
再运行 `refresh --all` 建立锁；不存在 `version-lock update --all` 操作。

### 日常命令

- 用 `artifact-graph init` 生成或检查项目制品图配置。
- 用 `artifact-graph validate` 校验制品之间的链接。
- 用 `artifact-graph validate-review-result --file <path>` 校验 Review Result Protocol v1.0 文档。
- 用 `artifact-graph context` 或 `artifact-graph packet` 构建实现上下文。
- 项目通过 `statusViews` 映射状态后，可在 `query`、`context` 或 `packet` 上增加
  `--view current|planned|history|all`；不传参数时仍返回兼容旧版的未过滤图。
- 用 `artifact-graph impact --worktree` 只读查看变更影响，不刷新版本锁。
- 用 `artifact-graph coverage` 报告图健康、扫描映射以及行为与发布证据的评估边界；
  该命令不推断验证成功或已经发布。
- 用 `artifact-graph version-lock refresh` 和 `audit` 保持追溯关系及时更新。
- 版本锁覆盖实现/验证边（`locks`）和制品间关系（`artifactRelations`）。旧版 1.0 锁文件缺少
  `artifactRelations` 时视为空数组；首次启用时执行一次 `refresh --all` 建立完整关系基线。
- 用 `artifact-graph hooks install-git --hook all` 安装可选 Git hooks。
- 用 `artifact-graph restructure inspect` 和 `artifact-graph restructure plan` 检查与规划制品重组：
  这两步只读取入并确定性地编译你给出的映射，不写项目。

### 制品重组（拆分、跨文件移动与重编号）

`artifact-graph restructure` 把一份显式重组映射编译成可复核的候选计划，再以一次文件集合操作
应用。支持三类变换：`record-split`（记录物理拆分）、`identity-split`（编号身份拆分）与
`move-renumber`（跨文件移动与重编号）。能力边界、共同约束与每项验收标准的去向不由 CLI 决定；
编译器只把一份完整映射编译成计划并应用。

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

`plan` 输出阻断项、未解决项、候选内部问题与写集外的引用出现位置；只要存在阻断项或未解决项，
`applicable` 就不为真，不可应用的计划不得进入 `apply`。计划文档要保存到写集之外的普通目录：
恢复只依赖该文档，不依赖进程状态。

> **采用边界。** 文件集合写入能力的成熟度为 `candidate`。资格环境仅为 Darwin / arm64 / APFS；
> 其他平台返回不可用，不会降级成无事务写入。该能力以合作式写者为前提，且不自证该前提：
> `apply` 与 `prune-recovery` 必须给出 `--confirm-cooperative-writers`，缺失即拒绝写入；
> 恢复必须用 `recover` 并同时给出 `--confirm-all-participants-stopped` 与
> `--confirm-exclusive-maintenance`。恢复材料默认保留，只有显式 `prune-recovery` 才清理。
> 不承诺全平台事务保证。`inspect` 与 `plan` 不写项目。

### 制品删除或拆分后的孤立锁清理

当制品、源码文件或追溯边不再存在时（常见于制品删除或拆分之后），对应的版本锁会成为
孤立锁。`version-lock refresh` 默认保留孤立锁，让清理始终是一次显式、可审查的操作：

```bash
# pnpm
pnpm exec artifact-graph version-lock refresh --all --remove-orphans --format markdown
git diff artifacts/traceability-version-lock.json   # 审查被移除的锁条目
git add artifacts/traceability-version-lock.json    # 暂存后重新提交

# npm
npx artifact-graph version-lock refresh --all --remove-orphans --format markdown
git diff artifacts/traceability-version-lock.json
git add artifacts/traceability-version-lock.json
```

只清理一次改号产生的那条边时，点名它，不要整体清扫：

```bash
pnpm exec artifact-graph version-lock refresh --changed-only --worktree --remove-orphan-edge <edgeId>
```

`--remove-orphan-edge` 可重复指定，只对当前仍是孤儿的边生效，并与 `--remove-orphans` 互斥
（同时给出即被拒绝）。点名仍是活边的边会被拒绝且不删除；同一锁文件里既有的孤立锁保持不变。

`version-lock audit` 会把结构性孤立锁和陈旧哈希标记为阻断问题，并输出上述解决步骤；
runner liveness 发现（测试文件不再处于任何已配置 runner 的激活范围）只是警告，不会阻断。

## Review Result Protocol

包内发布了 `schemas/review-result.schema.json`，以及配套的 TypeScript 类型和 `validateReviewResult`
校验 API。该协议与具体项目无关，覆盖 review、repair、批次证据、findings、metrics 和
fail-closed decision；非法字段和语义违规都会以稳定的 JSON path 报告。协议拒绝未知顶层字段；
`attempt` 取值仅限 1–3；成功接受必须带 `producer`；`PASS`/`PASS_WITH_RESIDUAL_MINOR` 不允许
存在未关闭的 `block` finding。独立的 repair 复审可以记录 `acceptance.reviewer` 和
`acceptance.source_result`，validator 会拒绝 repair producer 自我接受。
JSON Schema 无法比较跨对象的字段值，因此调用方还必须运行语义 validator；稳定身份由
`executor + name` 构成，`skill` 只是附加元数据，不能用来证明独立性。

## 文档导航

- [INSTALL.md](INSTALL.md) — 详细安装指南与 pnpm 10+ 配置
- [CHANGELOG.md](CHANGELOG.md) — 版本发布历史
- [CONTRIBUTING.md](CONTRIBUTING.md) — 贡献指南
- [SECURITY.md](SECURITY.md) — 安全策略

## 相关项目

如果需要 Codex / Claude Code 技能来引导制品链的接入、初始化和日常维护，请使用
[`artifact-chain-assistant`](https://github.com/ifoohoo/artifact-chain-assistant)。

## 开源协议

Apache-2.0。详见 [LICENSE](LICENSE)。
