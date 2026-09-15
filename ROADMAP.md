# 路线图

## 阶段一:Foundation(地基)

- [x] 仓库初始化与 SSH 推送配置
- [x] 基础文档(README / ARCHITECTURE / ROADMAP / docs/)
- [ ] `experiments/` 首批 3 个示例
- [ ] `prompts/` 首批 5 个模板
- [ ] `.gitignore` 完善
- [ ] `scripts/new-exp.sh` 实验脚手架

## 阶段二:Templates(模板)

- [ ] Python LLM 调用模板(OpenAI / Anthropic)
- [ ] Agent workflow 模板(LangGraph / CrewAI)
- [ ] Streamlit demo 模板
- [ ] Prompt 评测模板

## 阶段三:Tooling(工具)

- [ ] 仓库级 pre-commit(ruff / prettier)
- [ ] CI:lint + 基础 smoke test
- [ ] 标签化的实验元数据(`.exp-meta.json`)
- [ ] 简单 search index(`rg` 友好)

## 阶段四:Sharing(分享)

- [ ] 每个实验自动渲染 README 到 GitHub Pages
- [ ] Prompt 模板的版本管理与 diff
- [ ] 周报自动生成(基于 commits 与 notes)

## 非目标

- 不做生产化部署
- 不引入数据库/缓存等基础设施
- 不强制统一框架