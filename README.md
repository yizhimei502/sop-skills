# SOP Skills

A股量化项目沉淀的 3 个 SOP 工作流 skill，通用各 agent 工具，可直接放入任意 agent 的 skills 目录引用。

## 安装

将 `skills/` 下各 skill 目录拷贝到你使用的 agent 的 skills 目录：

- 任意 agent 工具的 skills 目录（如 `~/.agents/skills/`）

## 包含的 3 个 SOP

| Skill | 作用 | 输入 → 输出 |
|-------|------|-------------|
| **sop-dev-workflow** | 通用软件开发流程（从零造） | 需求 → 设计文档 + 源码 + 四层测试 + 用户手册 |
| **sop-code-analyze** | 代码分析（只读逆向） | 已有代码 → 架构文档 + 模块设计 + 注册表 + 分析报告 |
| **sop-code-normalize** | 代码规范化改造（克隆重构） | 已有代码 + 蓝图 → 对齐 dev-workflow 标准的完整工程副本 |

## 三者流水线关系

```
sop-dev-workflow       从零造，定义"标准形态"
      ▲
      │ 对齐目标
sop-code-normalize     在副本上改造，补齐对齐
      ▲
      │ 复用蓝图
sop-code-analyze       只读逆向，产出蓝图
```

完整链路：`原仓库 → sop-code-analyze → <repo>-analyze-sop（蓝图）→ sop-code-normalize → <repo>-normalize-sop（标准工程）→ 之后新增功能走 sop-dev-workflow`

## 设计要点

- **sop-dev-workflow**：5 角色协作（架构师/设计师/评审员/开发/测试）、契约驱动、闭卷考试评审、四层测试、JSON+MD 双格式报告、Conventional Commits。
- **sop-code-analyze**：只读逆向，4+3 角色（扫描→提取→核查→提炼 + ABC 验证闭环），置信度标注（confirmed/inferred/unknown），产出独立 `-analyze-sop/` 目录。
- **sop-code-normalize**：克隆副本改造（原仓库/蓝图只读），L1 结构重构 + L2 接口规范化 + 契约审计 + 基线比对 + 四层测试补齐，产出对齐 dev-workflow 的工程副本。

## 目录结构

```
skills/
├── sop-dev-workflow/
│   ├── SKILL.md            # 主规范
│   ├── prompts/            # 5 角色 prompt
│   ├── templates/          # 架构/设计/用户手册模板
│   └── schemas/            # review / bug / qa 报告 schema
├── sop-code-analyze/       # 同上结构
└── sop-code-normalize/     # 同上结构
```

## License

MIT
