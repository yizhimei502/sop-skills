---
name: code-normalize-sop
description: >
  代码规范化改造 Skill — 将已有代码仓库改造为对齐 dev-workflow-sop 标准的完整工程。
  TRIGGER when the user mentions: normalize legacy code, refactor to standard structure,
  规范化改造、把旧代码整理成标准工程、对齐 SOP、目录重构、补齐测试和文档。
  Also trigger when the user has a code-analyze-sop blueprint and wants to restructure the repo
  to dev-workflow-sop standard. Requires a prior code-analyze-sop run (blueprint input).
---

<!-- 本文件基于 code-normalize-sop.md v1.0 生成 -->

# code-normalize-sop — 代码规范化改造 Skill

## 一、核心定位

- **名称**：`code-normalize-sop`
- **输入**：
  - 原仓库 `<repo>/`（只读，改造对象）
  - 蓝图 `<repo>-analyze-sop/`（code-analyze-sop 产物，复用）
- **输出**：`<repo>-normalize-sop/` —— 规范化后的完整工程副本
- **核心原则**：
  - **原仓库永不修改**（与 code-analyze-sop 一致）
  - **最终产物对齐 dev-workflow-sop 标准形态**
- **北极星**：改造终点是"完整工程对齐"——目录结构、模块注册表、设计文档、四层测试、报告双格式、用户手册全部就位，后续可无缝进入 dev-workflow-sop 正常流程

### Skill 流水线关系

```
dev-workflow-sop      从零造，定义"标准形态"
      ▲
      │ 对齐目标
code-normalize-sop    在副本上改造，补齐对齐
      ▲
      │ 复用蓝图
code-analyze-sop      只读逆向，产出蓝图
```

完整链路：

```
原仓库
  │  code-analyze-sop（只读）
  ▼
<repo>-analyze-sop/   ← 蓝图：注册表 + design docs + normalization_hints
  │  code-normalize-sop（克隆改造）
  ▼
<repo>-normalize-sop/ ← 对齐 dev-workflow-sop 的完整工程
  │  之后
  ▼
新增功能 → 走 dev-workflow-sop 正常流程
```

---

## 二、产出位置（强制）

```
<repo>/                     ← 原仓库，纹丝不动
<repo>-analyze-sop/         ← 蓝图，只读
<repo>-normalize-sop/       ← 规范化副本（git clone 自原仓库，保留完整历史）
```

**副本获取方式：`git clone <repo> <repo>-normalize-sop/`**

- 保留原仓库完整 git 历史与分支
- 改造在副本的 `main` 分支上进行，每步独立 commit
- 将来用户决定替换时，副本的 `main` 直接 merge 回原仓库
- 改造出问题可 `git revert` 单独回滚

**蓝图复用方式**：直接读取 `<repo>-analyze-sop/`，只读引用，不改动它。

---

## 三、对齐目标（dev-workflow-sop 标准形态）

改造完成后，`<repo>-normalize-sop/` 必须呈现以下结构：

```
<repo>-normalize-sop/
├── .gitignore                          # 对齐 dev-workflow-sop 强制内容
├── MODULE_REGISTRY.json                # 对齐注册表 Schema
├── README.md
├── /docs
│   ├── architecture.md
│   ├── user_manual.md
│   ├── /design/module_<X>_design.md
│   └── /archive
├── /src/<模块名>/
├── /tests/{smoke,unit,contract,acceptance}/
├── /configs/{dev,prod}.env.example
├── /scripts/migrations
├── /build/
└── /reports/{reviews,qa}/{json,md}/
```

### 对齐检查清单（改造完成时逐项核对）

| # | 检查项 | 达标标准 |
|---|--------|----------|
| 1 | 目录结构 | 与上表一致，无多余顶层目录 |
| 2 | MODULE_REGISTRY.json | 每个模块有 status/depends_on/responsibility/src_path/design_doc |
| 3 | src/ 顶层目录 | 严格匹配注册表 src_path，无越权目录 |
| 4 | 设计文档 | 每个模块有 design doc，接口签名完整 |
| 5 | 四层测试 | smoke + unit + contract + acceptance 齐备 |
| 6 | 测试报告 | qa_final_report.json + .md 双格式，结论 PASS |
| 7 | 用户手册 | docs/user_manual.md 存在 |
| 8 | Git 提交 | 符合 dev-workflow-sop 提交规范 |

---

## 四、改造分级

| 级别 | 改什么 | 风险 | 默认 |
|------|--------|------|------|
| **L1 结构重构** | 按注册表重组 src 目录 + 改 import | 低（机械） | ✅ 默认全做 |
| **L2 接口规范化** | 补类型注解、docstring、统一命名 | 中（不改逻辑） | 按 hints 逐条做 |
| **L3 代码拆分** | 大函数拆分、模块拆分 | 高（可能引 bug） | ⚠️ 逐条需用户确认 |

> **循环 import 处理**：analyze 标记的 `hidden_couplings`（循环 import、隐式耦合）**只提示不改**，写入交付报告。拆循环属于 L3 深度重构，需用户明确要求才做。

---

## 五、完整流程定义

| 阶段 | 主导者 | 输入 | 具体动作 | 产出 |
|------|--------|------|----------|------|
| **Phase 0 准备** | 前置检查 | 原仓库 + 蓝图 | 1. `git clone` 原仓库到 `<repo>-normalize-sop/`<br>2. **蓝图时效校验**：analyzed_at 对应 commit vs 当前 HEAD，不一致 → 警告或重跑 analyze<br>3. 检查蓝图完整性：注册表、design docs、hints 齐备<br>4. 原仓库 git 状态检查（只读） | 副本仓库 + 蓝图校验结论 |
| **Phase 1 结构重构**（L1） | 重构者 | 蓝图注册表 + 副本源码 | 1. 按 `src_path` 重组 src 目录<br>2. 同步修改 import 路径<br>3. 修改测试/脚本/配置中的旧路径引用<br>4. 补全 SOP 骨架（.gitignore、/configs、/docs、/reports）<br>5. 每模块独立 commit | 重构后的 src + 骨架目录 |
| **Phase 2 接口规范化**（L2） | 重构者 | normalization_hints + design docs | 1. 按 hints 逐条补类型注解、docstring<br>2. **`[inferred]` 接口先确认**：向用户确认实际类型后才补，绝不按猜测写死<br>3. 统一命名规范<br>4. 每个 hint 独立 commit | 规范化后的源码 |
| **Phase 3 契约对齐验证** | 审计员 | 改造前后源码 + design docs | 1. **基线比对**：改造前记录测试输出（无测试则写冒烟锚点）<br>2. 改造后复跑基线，diff 输出<br>3. **契约审计**：接口签名是否与 design docs 一致<br>4. 不一致 → 阻断并修复 | 契约审计报告 + 基线比对结果 |
| **Phase 4 测试补齐** | 测试Agent | 全部 design docs + 副本源码 | 1. 冒烟测试补齐（开发Agent）<br>2. **单元/契约/验收只补核心路径**（主流程、关键接口、主要异常分支，不做全量 ≥80% 覆盖）<br>3. 按 dev-workflow-sop Phase 5 流程执行<br>4. 产出 qa_final_report.json/.md | tests/ 四层 + 测试报告 |
| **Phase 5 文档与交付** | 核查者 | 全部产物 | 1. 生成/更新 architecture.md、user_manual.md（复用 analyze 的 design docs 和 README.patch.md）<br>2. 跑通对齐检查清单<br>3. 产出交付总结（JSON + MD） | 对齐的完整工程 + 交付报告 |

> 注：normalize 验证采用**契约审计 + 基线比对**（针对"代码改了没坏"）。不复用 ABC 验证闭环——复用文档已经过 analyze 验证，改造重点是行为不变。

---

## 六、安全网：行为不变验证

```
改造前（Phase 3 起点）:
  1. 跑现有测试（如有）→ 记录通过率/输出
  2. 没有测试 → 先写关键路径冒烟锚点，记录输出

改造后（Phase 3 终点）:
  1. 复跑基线 → diff 输出
  2. 输出不一致 → 阻断，回滚该任务
  3. 契约审计：L1/L2 不应改变对外接口签名

验收:
  4. 全量测试通过 + 对齐检查清单全绿 → 交付
```

---

## 七、复用 code-analyze-sop 产物的清单

| 蓝图产物 | 在 normalize 中的用途 |
|----------|----------------------|
| `MODULE_REGISTRY.json` | Phase 1 目标目录结构 |
| `docs/design/module_<X>_design.md` | Phase 1 契约依据 + Phase 4 测试依据 + Phase 5 文档复用 |
| `analyze_report.json` 的 `normalization_hints` | Phase 2 改造任务清单 |
| `analyze_report.md` 的"需人工复核" | Phase 2 待确认的 `[inferred]`/`[unknown]` 接口 |
| `README.patch.md` | Phase 5 更新 README 的内容素材 |

---

## 八、错误处理与红线

| 红线 | 说明 |
|------|------|
| **原仓库只读** | 绝不写入 `<repo>/` 任何文件 |
| **蓝图只读** | 绝不修改 `<repo>-analyze-sop/`，只读引用 |
| **L1/L2 不改接口签名** | 接口签名变更走红色路径，需用户确认 |
| **不按猜测补注解** | `[inferred]`/`[unknown]` 接口必须用户确认后才补 |
| **行为不变** | 基线比对不一致 → 阻断，不得带病交付 |
| **独立提交** | 每个 L1 模块 / 每个 hint 独立 commit，符合 dev-workflow-sop 提交规范 |
| **L3 需确认** | 代码拆分（L3）逐条用户确认后执行 |
| **循环 import 只提示** | 隐式耦合/循环 import 不擅自拆分，写入交付报告 |

---

## 九、渐进式改造

支持 `--module <X>` 只改造单个模块，用于大仓库分批推进：

```
code-normalize-sop <repo> --module module_a
```

- 未指定模块 → 按蓝图 development_order 全量改造
- 已改造完成的模块标记 `status: normalized`，跳过
- 增量续跑不重复已完成的改造

---

## 十、执行指南

### 模板与 Schema 使用

| 用途 | 文件 |
|------|------|
| 架构文档 | `templates/architecture.md.template` |
| 设计文档 | `templates/design.md.template` |
| 用户手册 | `templates/user_manual.md.template` |
| 契约审计报告 | `schemas/contract_audit_schema.json` |
| 交付总结 | `schemas/delivery_report_schema.json` |
| 重构者 Prompt | `prompts/refactorer.md`（Phase 1/2） |
| 审计员 Prompt | `prompts/auditor.md`（Phase 3） |
| 测试补齐 Prompt | `prompts/tester.md`（Phase 4） |
| 核查者 Prompt | `prompts/checker.md`（Phase 5） |

### 执行顺序

```
1. 确认蓝图存在（<repo>-analyze-sop/），没有则先跑 code-analyze-sop
2. git clone 原仓库 → <repo>-normalize-sop/
3. 加载 refactorer.md → Phase 1 结构重构
4. 加载 refactorer.md → Phase 2 接口规范化（hints，inferred 先确认）
5. 加载 auditor.md → Phase 3 契约对齐验证
6. 加载 tester.md → Phase 4 测试补齐（核心路径）
7. 加载 checker.md → Phase 5 文档与交付（对齐检查清单）
8. 产出交付总结，git snapshot commit
```

---

## 十一、与其他 Skill 的衔接

```
改造前: code-analyze-sop <repo>          → 产出蓝图
改造中: code-normalize-sop <repo>        → 产出对齐副本
改造后: dev-workflow-sop（在副本上新增功能 → 正常 Phase 1-5）
持续:   code-analyze-sop --check         → 定期核对副本文档与代码一致性
```

---

## 十二、参考文件索引

| 文件 | 路径 | 用途 |
|------|------|------|
| 架构文档模板 | `templates/architecture.md.template` | Phase 5 渲染架构文档 |
| 设计文档模板 | `templates/design.md.template` | Phase 5 复用/渲染设计文档 |
| 用户手册模板 | `templates/user_manual.md.template` | Phase 5 渲染用户手册 |
| 契约审计 Schema | `schemas/contract_audit_schema.json` | Phase 3 审计报告结构 |
| 交付总结 Schema | `schemas/delivery_report_schema.json` | Phase 5 交付报告结构 |
| 重构者 Prompt | `prompts/refactorer.md` | Phase 1/2 角色定义 |
| 审计员 Prompt | `prompts/auditor.md` | Phase 3 角色定义 |
| 测试补齐 Prompt | `prompts/tester.md` | Phase 4 角色定义 |
| 核查者 Prompt | `prompts/checker.md` | Phase 5 角色定义 |

---

**规范版本**：v1.0 | **最后更新**：2026-08-08
