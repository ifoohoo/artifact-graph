# artifact-graph

[English](README.md)

`artifact-graph` 是一个 Git 原生的制品图扫描与校验工具，面向 AI 编程工作流。
默认解析 Markdown。自定义类型可以声明 `format: json`，把 JSON 原文件读进同一张图。

它把项目里的需求、场景、设计、源码、测试和 version-lock 元数据连成一张图，
供 AI agent 在宣称实现完成之前，用确定性的本地命令检查上下文和追溯关系。

<!-- release-skill:capability:external-write-boundary -->
> **外部写入边界：** 安装 `artifact-graph` 不会修改项目或写入远端。`--help`、`doctor`、
> `validate`、`query`、`audit`、不带 `--conclusion-output` 的 `check-professional` 以及
> `read-proof` 等只读命令不会更改项目文件。`--conclusion-output` 只在明确的绝对路径
> 排他创建一份证明，不会覆盖已有证明。只有显式执行 `init`、`scan`、
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

### 自定义类型的 JSON 原文件

内置类型继续使用各自的专用解析。`format: json` 只写在项目自己注册的自定义类型上，同一项还要给出 `idField`。省略 `format` 时，文件仍按 Markdown 解析，扫描路径写成 `*.json` 也不改变这一规则。类型声明 `format: json` 之后，才按 JSON 读取。

`idField` 是根对象上的一个属性名。查找时使用这个完整名字，名字里的点号仍是名字的一部分。每个文件只接受一个 JSON 对象，并生成一个节点。编号取该属性中的字符串，空格和大小写按原文保留，再用该类型已有的 `idPatterns` 检查。

下面的输入会给出诊断，并且不生成节点：

- JSON 文本无法解析；
- 根值是数组、字符串、数字、布尔值或 `null`；
- 编号属性不存在，或者值不是非空字符串。

编号字符串存在，但不符合 `idPatterns` 时，节点仍然保留，同时报出诊断。

关系继续写在 `relationSemantics` 里，沿用原来的 `kind`、`fields`、`targetTypes` 和 `label`。
JSON 文件把 `fields` 中的点分字符串当成逐层属性名。
以 `capability.capabilityRef` 为例，解析先读根对象自己的 `capability`，再读下一层对象自己的 `capabilityRef`。

某一层没有这个属性时，这条可选关系不进入图。中间值不是对象，或者末端不是非空字符串，也不是非空字符串数组时，诊断写明原文件路径和字段名。数组中的每个合法字符串交给现有目标解析。对不上现有节点的引用，仍由原来的图校验报告。空数组不生成边。

点号只切开固定的属性名。`*`、筛选条件和 JSONPath 都不是路径语法；名为 `*` 的属性只按这个名字查找。数组里的记录留在原对象中。这一批读取编号，以及 `relationSemantics` 里已经声明的关系。标题、状态、编号拼接、前缀和改写都不在读取范围内。Markdown 字段仍按整个字符串查找，嵌套对象不会被点号拆开。诊断带原文件路径和字段名，行号记为 1。

项目可以把方法指向能力，把发行记录指向方法。下面的配置只示范这两个方向。

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
```

方法文件把一个能力编号放在嵌套属性里。

```json
{
  "methodId": "METH-1",
  "capability": { "capabilityRef": "CAP-1" }
}
```

发行文件用字符串数组保存多个方法编号。`method:METH-1` 交给现有的 `type:id` 解析。

```json
{
  "releaseId": "REL-1",
  "methods": ["METH-1", "method:METH-2"]
}
```

`scanArtifacts`、`validate` 和 `version-lock` 消费同一张图，锁按原文件字节计算。读到节点和已配置关系，只说明这些原文件进入了图。领域审核和发布事实需要各自的证据。

### 日常命令

- 用 `artifact-graph init` 生成或检查项目制品图配置。
- 用 `artifact-graph validate` 校验制品之间的链接。
- 用 `artifact-graph check-professional --root . --format json` 生成可复用的图专业证明。
  只有需要证明文件时才加 `--conclusion-output <绝对路径>`；该命令组合已有的
  `validate`、`version-lock audit` 和 `coverage`，不写图缓存。
- 用 `artifact-graph read-proof --proof-root <root> --proof <相对路径> --format json`
  阅读本族证明。`GRAPH_CLEAR` 且范围 complete 为 `pass`；发现问题、检查未完成或证明记载
  未能检查为 `not_pass`。缺文件、损坏、非本族或未知领域码为 `unavailable`。这与
  Review Result Protocol 不是同一合同。
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
  这两步只读取入并确定性地编译已经给出的映射，不写项目。

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
