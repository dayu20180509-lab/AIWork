# 约定

## 命名

| 类型 | 风格 | 示例 |
|---|---|---|
| 目录 | kebab-case | `agent-chatbot` |
| Python 文件 / 变量 | snake_case | `parse_response.py` |
| 类 | PascalCase | `class AgentRunner` |
| Markdown | kebab-case | `setup-guide.md` |
| 环境变量 | SCREAMING_SNAKE | `OPENAI_API_KEY` |

## 提交信息

遵循 Conventional Commits:

```
<type>(<scope>): <subject>

<body>

<footer>
```

- **type**: `feat` / `fix` / `docs` / `refactor` / `test` / `chore` / `exp`
- **scope**(可选):目录名,如 `exp(chatbot)`、`docs(setup)`
- **subject**:中文或英文均可,**不超过 50 字**

示例:

```
exp(chatbot): add streamlit ui
docs: clarify git workflow
fix(snippets/parse): handle empty input
```

## 代码风格

- Python: ruff + black,line length 100
- JS/TS: prettier,line length 100
- Markdown:不强制

## Git 工作流

- 主分支:`main`,受保护,只接受 fast-forward 或 squash merge
- 个人分支:`feat/<name>` / `fix/<name>` / `exp/<name>`
- 单次提交尽量保持原子性

## 敏感信息

API key、token、内部 URL **永不进 git**。使用:

- `.env`(仓库根目录,`.gitignore` 已忽略)
- `direnv` 自动加载
- 1Password CLI(`op://...`)用于团队共享

## 不强制

- 不强制测试覆盖率
- 不强制 type hint
- 不强制单一语言