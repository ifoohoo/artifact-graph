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

如果任一只读命令失败，先确认当前 Node.js 版本不低于 `22.0.0`，再按照
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

需要 Node.js `>=22.0.0`。pnpm 10+ 需要配置原生构建白名单，详见 [INSTALL.md](INSTALL.md)。

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

- 用 `artifact-graph init` 生成或检查项目制品图配置。
- 用 `artifact-graph validate` 校验制品之间的链接。
- 用 `artifact-graph validate-review-result --file <path>` 校验 Review Result Protocol v1.0 文档。
- 用 `artifact-graph context` 或 `artifact-graph packet` 构建实现上下文。
- 用 `artifact-graph version-lock refresh` 和 `audit` 保持追溯关系及时更新。
- 版本锁覆盖实现/验证边（`locks`）和制品间关系（`artifactRelations`）。旧版 1.0 锁文件缺少
  `artifactRelations` 时视为空数组；首次启用时执行一次 `refresh --all` 建立完整关系基线。
- 用 `artifact-graph hooks install-git --hook all` 安装可选 Git hooks。

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
