# 使用指南

## 添加一个新实验

1. 创建目录:

   ```bash
   mkdir -p experiments/<your-exp-name>
   cd experiments/<your-exp-name>
   ```

2. 初始化 README:

   ```markdown
   # <your-exp-name>

   ## 目的
   ...

   ## 运行
   ```bash
   python main.py
   ```

   ## 结果
   ...
   ```

3. 添加依赖(如 Python):

   ```bash
   uv init
   uv add openai
   ```

4. 编写代码并 commit:

   ```bash
   git add .
   git commit -m "exp(<name>): initial scaffold"
   ```

## 添加 prompt 模板

直接在 `prompts/` 下添加 `<purpose>.md`,例如:

```markdown
# Code Review

请审查以下代码,重点关注:

- 边界条件
- 错误处理
- 性能
...

<贴入代码>
```

## 跨实验复用代码

将通用代码移到 `snippets/`,作为独立可运行脚本,而不是 import 库。

## 数据

- 小样本:`experiments/<name>/fixtures/`
- 大数据:`data/`(gitignored),记录获取方式到实验 README

## 搜索

```bash
# 找所有 prompt
rg -l "^# " prompts/

# 找引用了 openai 的代码
rg "openai" experiments/ snippets/

# 找特定话题的笔记
rg -l "embedding" notes/
```

## 清理

- 实验完结不打算保留 → 归档到 `experiments/_archive/`,或直接删除(commit 历史可追溯)