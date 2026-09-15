# 环境搭建

## 前置要求

- macOS / Linux / WSL
- Git(SSH 已配置,见根目录文档)
- Python 3.11+ / Node 20+(按需)
- [uv](https://github.com/astral-sh/uv)(推荐)或 pip

## 克隆

```bash
git clone git@github.com:dayu20180509-lab/AIWork.git
cd AIWork
```

## 工具建议

| 工具 | 用途 |
|---|---|
| `uv` | Python 依赖与虚拟环境 |
| `direnv` | 自动加载 `.envrc` |
| `pre-commit` | Git hooks,自动 lint |
| `rg` (ripgrep) | 快速搜索 |
| `gh` | GitHub CLI |

安装示例(macOS):

```bash
brew install uv direnv ripgrep gh
brew install pre-commit
```

## 仓库级初始化

```bash
./scripts/init.sh   # 安装 pre-commit hooks 等(待实现)
```

## 编辑器

推荐 VS Code 或 Cursor。配置见 [tools.md](tools.md)。

## 下一步

- 阅读 [usage.md](usage.md) 学习如何添加实验
- 阅读 [conventions.md](conventions.md) 了解命名与提交风格