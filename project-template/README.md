# Project Template

这是通用项目开发规范模板。

版本：v0.1.0  
适配：Go 项目，默认使用 `kit/log`、`kit/http`、`kit/sql`  
更新时间：2026-07-03

## 文件说明

```text
AGENTS.md                 AI / Codex 执行规范
CLAUDE.md                 Claude Code 执行入口
doc/项目开发规范.md        开发者阅读的完整项目规范
doc/模板维护说明.md        模板维护流程和检查项
```

## 使用方式

新项目初始化建议使用 `kit` 项目生成工具，默认生成 `kit/log`、`kit/http`、`kit/sql` 项目骨架。

新项目创建后，将模板文件复制到项目根目录：

```text
AGENTS.md
CLAUDE.md
doc/项目开发规范.md
```

如果项目已有 README，可在 README 中增加规范入口：

```markdown
开发规范请查看 [doc/项目开发规范.md](doc/项目开发规范.md)，AI 执行规范请查看 [AGENTS.md](AGENTS.md)，Claude Code 入口请查看 [CLAUDE.md](CLAUDE.md)。
```

## 维护方式

模板维护请查看 [doc/模板维护说明.md](doc/模板维护说明.md)。
