# AIWork

> AI 编程工作空间 — 存放 AI 实验、脚本、模板与笔记

AIWork 是一个个人/团队的 AI 编程实验场,用于沉淀:

- LLM 调用脚本与 prompt 模板
- Agent / workflow 验证代码
- 数据处理与可视化小工具
- 学习笔记与最佳实践

## 快速开始

```bash
git clone git@github.com:dayu20180509-lab/AIWork.git
cd AIWork

# 进入特定实验
cd experiments/<your-experiment>

# 各子目录独立管理依赖
pip install -r requirements.txt   # 或 uv sync / npm install 等
```

## 目录结构

```
AIWork/
├── README.md           # 本文件
├── ARCHITECTURE.md     # 架构与目录组织约定
├── ROADMAP.md          # 路线图
├── docs/               # 详细文档
│   ├── setup.md
│   ├── usage.md
│   ├── conventions.md
│   └── tools.md
├── experiments/        # 实验代码(各自独立子目录)
├── prompts/            # prompt 模板集合
├── snippets/           # 可复用代码片段
├── notes/              # 学习笔记 / 调研记录
└── templates/          # 项目/脚本模板
```

## 工作流

1. 新建实验:在 `experiments/<name>/` 下创建子目录,自带 README 与代码
2. 编写代码、独立提交(commit message 遵循 Conventional Commits)
3. 完成后合并到 main

更多细节见 [docs/usage.md](docs/usage.md)。

## 贡献

欢迎添加实验、prompt、snippet。参考 [docs/conventions.md](docs/conventions.md)。

## 许可

待定