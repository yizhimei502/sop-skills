# Claude SOP Skills

A股量化项目沉淀的 3 个 Claude Code skill（SOP 工作流），按 Claude Code 插件/skill 标准结构组织，可一键安装。

## 安装

```bash
claude plugin install github:<你的用户名>/claude-sop-skills
```

或手动放入 `~/.claude/skills/`（本仓库 `.claude/skills/` 下各目录直接拷贝）。

## 包含的 3 个 SOP

| Skill | 作用 | 输入 → 输出 |
|-------|------|-------------|
| **dev-workflow-sop** | 通用软件开发流程（从零造） | 需求 → 设计文档 + 源码 + 四层测试 + 用户手册 |
| **code-analyze-sop** | 代码分析（只读逆向） | 已有代码 → 架构文档 + 模块设计 + 注册表 + 分析报告 |
| **code-normalize-sop** | 代码规范化改造（克隆重构） | 已有代码 + 蓝图 → 对齐 dev-workflow 标准的完整工程副本 |

## 三者流水线关系

```
dev-workflow-sop      从零造，定义"标准形态"
      ▲
      │ 对齐目标
code-normalize-sop    在副本上改造，补齐对齐
      ▲
      │ 复用蓝图
code-analyze-sop      只读逆向，产出蓝图
```

完整链路：`原仓库 → code-analyze-sop → <repo>-analyze-sop（蓝图）→ code-normalize-sop → <repo>-normalize-sop（标准工程）→ 之后新增功能走 dev-workflow-sop`

## 设计要点

- **dev-workflow-sop**：5 角色协作（架构师/设计师/评审员/开发/测试）、契约驱动、闭卷考试评审、四层测试、JSON+MD 双格式报告、Conventional Commits。
- **code-analyze-sop**：只读逆向，3+3 角色（扫描→提取→核查 + ABC 验证闭环），置信度标注（confirmed/inferred/unknown），产出独立 `-analyze-sop/` 目录。
- **code-normalize-sop**：克隆副本改造（原仓库/蓝图只读），L1 结构重构 + L2 接口规范化 + 契约审计 + 基线比对 + 四层测试补齐，产出对齐 dev-workflow 的工程副本。

## 目录结构

```
.claude/skills/
├── dev-workflow-sop/
│   ├── SKILL.md            # 主规范
│   ├── prompts/            # 5 角色 prompt
│   ├── templates/          # 架构/设计/用户手册模板
│   └── schemas/            # review / bug / qa 报告 schema
├── code-analyze-sop/       # 同上结构
└── code-normalize-sop/     # 同上结构
```

## License

MIT
