# 工具与服务

## 编辑器

### VS Code / Cursor

推荐扩展:

- Python(microsoft)
- Ruff(astral-sh.ruff)
- Even Better TOML
- Markdown All in One
- GitLens

`settings.json` 建议:

```json
{
  "python.defaultInterpreterPath": "~/.local/share/uv/python/cpython-3.11.*",
  "[python]": { "editor.defaultFormatter": "astral-sh.ruff" },
  "editor.formatOnSave": true
}
```

## Python 工具链

| 工具 | 替代 | 用途 |
|---|---|---|
| `uv` | pip + venv | 依赖与虚拟环境 |
| `ruff` | flake8 + black + isort | lint + format |
| `pyright` | mypy | 类型检查(可选) |

## Node 工具链

| 工具 | 用途 |
|---|---|
| `pnpm` | 依赖管理(优先于 npm) |
| `tsx` | 直接运行 TS |
| `prettier` | 格式化 |

## Git 增强

- `gh`:GitHub CLI,创建 PR、查看 issue
- `git-delta`:更友好的 diff
- `lazygit`:终端 TUI

## AI 服务

| 服务 | 配置 |
|---|---|
| OpenAI | `OPENAI_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |
| Google | `GOOGLE_API_KEY` |
| Cohere | `COHERE_API_KEY` |

通过 `direnv` + `.envrc` 加载:

```
export OPENAI_API_KEY=...
export ANTHROPIC_API_KEY=...
```

## 日志与可观测

- 实验内:简单 `print` / `logging` 即可
- 长时间运行:考虑 `wandb` / `mlflow`(单独 opt-in)

## 不依赖

- Docker(按需,各实验自带 Dockerfile)
- 数据库(数据走文件系统)
- 缓存服务(本地)