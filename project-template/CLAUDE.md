# CLAUDE.md

## 执行规范

本项目开发规范以 `AGENTS.md` 为准。

Claude Code 执行任何代码修改前，必须先阅读：

- `AGENTS.md`
- `doc/项目开发规范.md`

## 核心约束

- 默认使用中文回复
- 修改前先判断任务复杂度
- 复杂任务必须先出方案，等待确认后再自动开发、测试、提交并推送
- 按 `api -> service -> repo -> model` 分层
- Go 项目默认使用 `kit/log`、`kit/http`、`kit/sql`
- 禁止擅自替换日志库、HTTP 框架、数据库访问库或 ORM
- 禁止硬编码密钥、Token、数据库密码
- 未经单独确认禁止部署
- 修改后执行必要测试；测试通过后按规范执行 `git commit` 和 `git push`；每次修改后说明修改文件、影响范围、是否需要联动修改、是否已更新文档

## 文档入口

- AI 执行规范：`AGENTS.md`
- 完整开发规范：`doc/项目开发规范.md`
