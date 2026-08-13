---
name: code-analyze-sop
description: >
  代码分析 Skill — 逆向工程已有代码仓库，提取架构与接口，输出设计文档和分析报告。
  TRIGGER when the user mentions: analyze a codebase, understand existing code, generate docs from code,
  分析代码仓库、看看这个项目做了什么、把接口签名列出来、文档和代码一致吗、代码考古、逆向理解已有项目。
  Also trigger when the user gives you an existing repo path and asks what it does / to produce docs /
  to check doc-code consistency. Do NOT use this skill for new-project development (use dev-workflow-sop).
---

<!-- 本文件基于 code-analyze-sop.md v1.0 生成 -->

# code-analyze-sop — 代码分析 Skill

## 一、核心定位

- **名称**：`code-analyze-sop`
- **输入**：已有代码仓库路径
- **输出**：架构文档、模块设计文档、模块注册表、分析报告（含置信度评分）
- **核心原则**：**只读**——绝不修改原仓库任何源文件、目录结构或文件名
- **产出标注**：每个结论标记置信度 → `[confirmed]`（源码明确）| `[inferred]`（推断）| `[unknown]`（无法确定）

### 与 `dev-workflow-sop` 的关系

```
dev-workflow-sop              code-analyze-sop
─────────────────              ────────────
需求 → 设计 → 代码              代码 → 提取 → 文档
设计先行，编码为后              代码已有，文档后补
接口签名：设计师定义             接口签名：从代码提取
5 角色全流程                    3+3 角色（扫描→提取→核查 + ABC验证）
必改代码                        绝不该代码
```

## 二、产出位置（强制）

**绝不写入原仓库**。分析产出放在原仓库同级，加 `-analyze-sop` 后缀：

```
/path/to/my-project/                  ← 原仓库，纹丝不动
/path/to/my-project-analyze-sop/      ← 分析产出，同级独立
```

产出内部结构：

```
<项目名>-analyze-sop/
│
├── /docs
│   ├── architecture.md                 # Phase C 产出（模块职责边界 + 依赖 DAG）
│   ├── /design
│   │   └── module_<X>_design.md       # Phase C 产出（含从代码提取的接口签名）
│   └── /archive                        # 如有旧文档，原样移入此处
│
├── MODULE_REGISTRY.json                # Phase C 产出
├── README.patch.md                     # Phase C 产出（README 更新建议，用户自行拷贝）
│
└── /reports
    └── /analyze
        ├── /json
        │   ├── analyze_report.json     # Phase C 产出（机读分析报告）
        │   └── check_report.json       # --check 模式产出（机读差异报告）
        └── /md
            ├── analyze_report.md       # Phase C 产出（人读分析报告）
            └── check_report.md         # --check 模式产出（人读差异报告）
```

### 产出目录的 Git 管理

分析完成后，对产出目录做轻量 Git 管理：

```bash
cd <项目名>-analyze-sop/
git init
git add -A
git commit -m "analyze: 初始代码分析完成（可信度 XX%）"
```

不做分支策略、不强制多 commit。仅保留一个 snapshot，便于后续 `code-normalize` 接手、重新分析时 diff 变化、`--since` 增量分析定位基准点。

### 对原仓库 README 的处理

`code-analyze-sop` 不写入原仓库。如需更新 README，产出 `README.patch.md` 包含建议内容，用户自行 copy-paste。

---

## 三、角色定义（3+3 角色）

### 3.1 分析流水线（Phase A-C）

| 角色 | 职责 | 输入 | 产出 |
|------|------|------|------|
| **扫描者**（Scanner） | 仓库全景扫描：技术栈、目录拓扑、模块边界、外部依赖。**按功能粒度划分模块**，为每个模块边界写判定理由 | 目录结构 + 配置文件 + import 语句 | 技术栈清单、模块候选列表（含边界理由）、依赖图谱、跳过文件清单 |
| **提取者**（Extractor） | 接口签名提取：逐文件深入，标注置信度 | Phase A 模块列表 + 源码 | 接口签名清单（含置信度+来源）、数据结构清单、异常清单 |
| **核查者**（Auditor） | 文档生成：综合 Phase A + B 数据，渲染模板 | 全部分析数据 + 已有文档（如有） | `/docs/architecture.md` + `/docs/design/*.md` + `MODULE_REGISTRY.json` + 分析报告 |

### 3.2 ABC 验证闭环（Phase D）

| 角色 | 能读什么 | 严禁读 | 做什么 |
|------|---------|--------|--------|
| **Agent B**（出题者） | 源码 + 分析产出 | — | 生成 N 道"闭卷考题" + 标准答案（answer key） |
| **Agent C**（答题者） | **仅**分析产出（`-analyze-sop/` 下所有文件） | **严禁**读原仓库源码 | 闭卷答题。答不上 / 答错 → 文档有 gap |
| **评估员** | — | — | 逐条比对 C 的答案 vs B 的 answer key |

**验证核心理念：信息隔离。** C 能答对的题 = 文档足够完整；C 答不上 = 文档确实缺信息。同等级模型下依然成立。

---

## 四、完整流程定义

| 阶段 | 主导者 | 输入 | 具体动作 | 产出 |
|------|--------|------|----------|------|
| **Phase A** | 扫描者 | 仓库根目录路径 | 1. **错误检查**：路径不存在 → 终止；空仓库 → 终止<br>2. **Monorepo 检测**：满足 ≥2 条件 → 多项目分析模式<br>3. 识别技术栈<br>4. 绘制目录拓扑<br>5. **按功能粒度划分模块**，写边界理由，不确定列入"边界争议"<br>6. 提取外部依赖清单<br>7. 查找已有文档<br>8. 识别测试框架<br>9. 标记无法解析文件到"跳过文件清单" | 技术栈清单、目录拓扑图、模块候选列表（含边界理由）、依赖图谱、已有文档索引、跳过文件清单 |
| **Phase B** | 提取者 | Phase A 产出 + 源码 | 1. 逐模块提取公开函数/类/方法签名<br>2. 标注置信度<br>3. 提取模块职责（`__init__.py` docstring→confirmed；公共函数 docstring 聚合→inferred；目录名→inferred）<br>4. 提取异常类型<br>5. 提取数据结构<br>6. 梳理调用关系<br>7. 识别隐式耦合<br>8. 标记无法解析依赖 → `[unresolved_dependency]` | 接口签名清单（含置信度+来源）、模块职责（含来源）、数据结构清单、异常清单、调用关系矩阵 |
| **Phase C** | 核查者 | Phase A + B 全部分析数据 | 1. 渲染 `architecture.md`<br>2. 渲染每个模块 `design.md`<br>3. 生成 `MODULE_REGISTRY.json`<br>4. 生成分析报告（可信度评分 + 复核清单）<br>5. 产出 `README.patch.md`<br>6. **交叉校验**：Scanner 的依赖结论 vs Extractor 的 import 证据，不一致入 `hidden_couplings` | `architecture.md`、`module_<X>_design.md`、`MODULE_REGISTRY.json`、`analyze_report.json/.md`、`README.patch.md` |
| **Phase D** | Agent B + C + 评估员 | **B**：源码 + 分析产出<br>**C**：仅分析产出，严禁源码 | ABC 验证循环（见下章） | 验证轮次报告，写入 `analyze_report` 的 `verification` 字段 |

### 场景路由

| 场景 | 触发语示例 | 路由 |
|------|-----------|------|
| 全量分析 | "帮我分析一下这个仓库"、"看看这个项目做了什么" | Phase A → B → C 全量 |
| 只出架构图 | "这个项目的模块依赖是什么" | Phase A（仅扫描） |
| 只提取接口 | "把 API 签名列出来" | Phase A → B（跳过文档渲染） |
| 一致性检查 | "文档和代码一致吗"、"check 一下" | `--check` 模式 |
| 增量分析 | "代码更新了，重新分析一下" | `--since <commit>` 增量模式 |
| 子项目分析 | "只分析 packages/backend" | `--scope <path>` 指定子项目 |

### 错误处理策略

| 级别 | 情况 | 处理 |
|------|------|------|
| **终止** | 仓库路径不存在 | 直接报错退出 |
| **终止** | 仓库完全为空（0 个文件） | 直接报错退出 |
| **跳过** | 单个文件语法错误 | 跳过，记录到"跳过的文件"清单，继续分析 |
| **跳过** | 二进制文件（图片、`.so`、`.pyc`） | 忽略 |
| **跳过** | 空文件 | 忽略 |
| **标记** | import 了但不存在的模块 | 标记为 `[unresolved_dependency]`，继续分析 |

---

## 五、置信度体系

| 源码证据 | 标注 | 说明 |
|----------|------|------|
| 有类型注解 + 文档字符串 | `[confirmed]` | 源码直接声明，可信 |
| 有类型注解，无文档 | `[confirmed]` | 类型已知，语义需读代码 |
| 无类型注解，但调用模式清晰 | `[inferred]` | 从调用处推断 |
| 无类型注解，部分可推断 | `[inferred]` | 部分已知 |
| 无类型注解 + 动态参数 + 无文档 | `[unknown]` | 无法推断 |

**可信度评分** = `confirmed 条目数 / 总条目数 × 100%`

| 评分区间 | 等级 | 含义 |
|----------|------|------|
| ≥ 80% | 🟢 高 | 文档基本可靠，少数项需人工确认 |
| 60%-79% | 🟡 中 | 建议人工复核 [inferred] 条目 |
| < 60% | 🔴 低 | 大量推断项，强烈建议人工全面复核 |

> 可信度 < 60% → 在 `analyze_report.md` 顶部加醒目警告。每个 `[inferred]` 和 `[unknown]` 条目必须列在"需人工复核"表中。

---

## 六、ABC 验证闭环（Phase D 详细规范）

### 6.1 Agent B 出题规范

Agent B 读源码 + 分析产出，生成 N 道验证题。建议模型：成本较低的同等或弱模型（不做强制型号要求）。

考题类型：

| 类型 | 示例问题 | 校验目标 |
|------|----------|----------|
| 接口签名 | "模块 A 的 `get_user()` 接受哪些参数？各自类型？" | 签名提取完整性 |
| 调用关系 | "模块 A 依赖了哪些外部模块？列出具体 import 路径。" | 依赖图谱准确性 |
| 边界条件 | "`get_user()` 传入空字符串会抛什么异常？" | 异常和边界覆盖 |
| 职责描述 | "模块 C 的核心职责是什么？它不负责什么？" | 职责提取准确性 |
| 数据结构 | "`Order` 类有哪些字段？类型和含义？" | 数据结构提取完整性 |

每题附带 answer key（标准答案，从源码直接得出一一对应，不可主观）：

```json
{
  "question": "模块 weather_api 的 get_current_weather() 接受哪些参数？类型？默认值？",
  "type": "interface_signature",
  "required_facts": [
    "city_name: str（必填，无默认值）",
    "timeout: float（可选，默认 10.0）"
  ]
}
```

### 6.2 Agent C 答题规范

Agent C **只能阅读** `-analyze-sop/` 下的所有文件。**严禁**访问原仓库源码。

- 从文档中寻找答案
- 找不到 → 回答"我不知道"
- 不许推测、不许"大概可能是"
- "我不知道" → 文档有 gap → fail

### 6.3 评估标准

- **PASS**：C 的答案覆盖 ALL required facts（语义等价即可）
- **FAIL**：缺失任意一个 required fact，或 C 回答"我不知道"

**客观检查，不做主观判断。**

### 6.4 循环规则

```
Phase C 产出分析文档
       ↓
  [第 1 轮] B 出题 → C 答题 → 评估
       ↓
   100% 通过? ──YES──→ ✅ 验证完成
       │
      NO（哪怕差 1 题）
       │
       ▼
  打回 Agent A：给出失败题目 + B 的答案 + C 的答案
  Agent A 补文档（不许看 B 的考题，只能根据 gap 描述修补）
       │
       ▼
  [第 2 轮] B 出**新题**（不重复旧题）→ C 答题 → 评估
  [第 3 轮] 同上 → 评估
       │
   100% 通过? ──YES──→ ✅ 验证完成
       │
      NO ──→ ⚠️ 不再循环。残留失败写入报告，用户裁决。
```

**硬规则：最多 3 轮，绝不提前终止。99% 通过也不行——差一题就是 fail。**

### 6.5 验证结果写入报告

`analyze_report.json` 增加 `verification` 字段：

```json
"verification": {
  "rounds": 2,
  "final_pass_rate": 100,
  "total_questions_asked": 18,
  "rounds_detail": [
    {"round": 1, "questions": 10, "passed": 7, "failed": 3},
    {"round": 2, "questions": 8, "passed": 8, "failed": 0}
  ],
  "unresolved_gaps": []
}
```

`analyze_report.md` 增加验证详情章节（轮次表 + 结论 + 未解决缺陷表）。

### 6.6 交叉校验（Phase C 附带，非循环）

Scanner 的依赖结论 vs Extractor 的 import 证据不一致 → 标记进 `hidden_couplings`，不参与 ABC 循环。

---

## 七、一致性检查模式（`--check`）

比对当前代码与已有设计文档，只出差异报告，不改任何文件。

| 差异类型 | 含义 | 建议动作 |
|----------|------|----------|
| `[ok]` | 代码与文档一致 | 无需处理 |
| `[missing_in_doc]` | 代码有、文档无 | 更新文档 |
| `[missing_in_code]` | 文档有、代码无 | 更新文档 |
| `[mismatch]` | 两者都有但签名不一致 | 确认以谁为准 |

文档来源自动检测，优先级递减：

1. `<项目名>-analyze-sop/docs/design/`（之前 code-analyze-sop 产出的）
2. `<项目>/docs/design/`（仓库自带）
3. 都没有 → 报错"没有可对比的设计文档，请先运行 code-analyze-sop"

非标准格式文档 → 跳过比对，报告末尾列"无法比对的文档"。

注意：`ok` 条目只纳入 `summary.ok` 计数，不逐条写入 `details`。

---

## 八、增量分析（`--since`）

```
code-analyze-sop /path/to/repo --since HEAD~10
```

增量模式流程：

1. `git diff <commit>..HEAD` 获取变更文件列表
2. 只重新扫描变更的文件
3. 增量更新设计文档和注册表（新签名追加，已删除标记 archived）
4. 未变更的模块**原样保留**（包括人工修改过的内容）
5. 重新计算可信度评分

前提：产出目录已在 Git 管理下，且存在上一份分析报告。

---

## 九、Monorepo 多项目分析模式

检测条件（满足 ≥ 2 个即判定为 monorepo）：

1. 顶层存在 `packages/`、`apps/`、`services/` 等目录
2. 多个包管理器文件共存（`package.json` + `requirements.txt` + `go.mod` 等）
3. 存在 workspace/workspaces 配置

判定后**每个子项目独立分析**（`packages/frontend-analyze-sop/` 等），顶层产出 `monorepo_index.md` 汇总。

---

## 十、执行指南

### 模板与 Schema 使用

| 用途 | 文件 |
|------|------|
| 架构文档 | `templates/architecture.md.template` |
| 模块设计文档（提取版） | `templates/analyze-design.md.template` |
| 分析报告 | 遵循 `schemas/analyze_report_schema.json` |
| 差异报告 | 遵循 `schemas/check_report_schema.json` |
| 扫描者 Prompt | `prompts/scanner.md`（Phase A） |
| 提取者 Prompt | `prompts/extractor.md`（Phase B） |
| 核查者 Prompt | `prompts/auditor.md`（Phase C） |
| Agent B Prompt | `prompts/agent-b.md`（出题） |
| Agent C Prompt | `prompts/agent-c.md`（答题） |

### 执行顺序

```
1. 定位仓库 → 确定产出目录（-analyze-sop 后缀）
2. 加载 scanner.md → Phase A
3. 加载 extractor.md → Phase B
4. 加载 auditor.md → Phase C
5. 加载 agent-b.md + agent-c.md → Phase D（ABC 验证循环）
6. 生成 analyze_report.json/.md（含 verification 字段）
7. git init 产出目录 + snapshot commit
```

---

## 十一、与其他 Skill 的衔接

```
code-analyze-sop 产出的 MODULE_REGISTRY.json
       +
code-analyze-sop 产出的 /docs/design/*.md
       +
code-analyze-sop 产出的 normalization_hints（analyze_report.json）
       ↓
   作为 code-normalize（未来 Skill 3）的输入
       ↓
   code-normalize 按 SOP 重构目录、补齐测试
```

---

## 十二、参考文件索引

| 文件 | 路径 | 用途 |
|------|------|------|
| 架构文档模板 | `templates/architecture.md.template` | Phase C 渲染架构文档 |
| 设计文档模板（提取版） | `templates/analyze-design.md.template` | Phase C 渲染模块设计文档 |
| 分析报告 Schema | `schemas/analyze_report_schema.json` | 机读分析报告结构 |
| 差异报告 Schema | `schemas/check_report_schema.json` | --check 差异报告结构 |
| 扫描者 Prompt | `prompts/scanner.md` | Phase A 角色定义 |
| 提取者 Prompt | `prompts/extractor.md` | Phase B 角色定义 |
| 核查者 Prompt | `prompts/auditor.md` | Phase C 角色定义 |
| 出题者 Prompt | `prompts/agent-b.md` | Phase D Agent B |
| 答题者 Prompt | `prompts/agent-c.md` | Phase D Agent C |

---

**规范版本**：v1.0 | **最后更新**：2026-08-08
