# 架构

AIWork 作为「工作空间」而非单一应用,其「架构」主要体现为**目录组织、命名约定和模块边界**。

## 顶层布局

```
AIWork/
├── experiments/   # 实验性代码,可运行、可废弃
├── prompts/       # 纯文本 prompt 模板
├── snippets/      # 跨项目可复用的代码片段
├── notes/         # Markdown 笔记与调研
├── templates/     # 项目初始化模板
├── docs/          # 长期维护的文档
└── scripts/       # 仓库自身使用的辅助脚本
```

## 实验 (experiments)

每个实验是**独立子目录**,拥有自己的:

- `README.md` — 实验目的、运行方式、结果
- 依赖声明(`requirements.txt` / `package.json` / `pyproject.toml`)
- 代码与数据(数据建议放子目录 `data/`,加进 `.gitignore`)

**禁止实验之间直接相互 import** — 如需复用,把代码提到 `snippets/` 或抽取为独立包。

## 命名约定

| 类型 | 风格 | 示例 |
|---|---|---|
| 目录 | kebab-case | `agent-streamlit-ui` |
| 文件(代码) | snake_case | `parse_response.py` |
| 文件(Markdown) | kebab-case | `setup-guide.md` |
| Prompt 文件 | `<用途>.md` | `code-review.md` |
| 类 | PascalCase | `class AgentRunner` |
| 环境变量 | SCREAMING_SNAKE | `OPENAI_API_KEY` |

## 依赖管理

不设顶层依赖文件。每个实验/snippet 自带依赖,使用:

- Python: `pyproject.toml` + `uv` / `poetry`
- Node: `package.json`
- 其他: 自带 `requirements.txt` 等

**理由**:工作空间内代码异质,集中管理反而带来冲突。

## 数据

- 大文件、二进制、数据集:**不进 git**(`.gitignore` 覆盖)
- 小样本、prompt 用的 fixture:可入 git,放 `experiments/<name>/fixtures/`

## AI 服务配置

API key 等敏感信息**永不进 git**。通过环境变量或本地 `.env`(被 `.gitignore`)注入。

推荐:`direnv` + `.envrc`,或 `1Password CLI`。

## 文档同步

- 仓库结构变更 → 更新本文件
- 新增约定 → 更新 `docs/conventions.md`
- 新增工具/服务 → 更新 `docs/tools.md`

## 不做的事

- 不引入 monorepo 构建系统(turborepo / nx 等)— 复杂度不匹配
- 不强制单一语言
- 不做强制代码 review 流程(本仓库以个人为主)