# 在本项目中使用 Spec Kit

## 已接入的内容

- Spec Kit 版本：`1.4.4`，使用官方 `specify-cli` 发布包。
- Codex 集成：项目技能模式，脚本类型为 Bash；未启用额外生命周期事件。
- `.agents/skills/`：9 个 `speckit-*` 技能。
- `.specify/`：模板、脚本、集成清单、项目原则模板及标准工作流。
- `AGENTS.md`：根据现有代码整理的项目结构、运行上下文和验证说明。

Spec Kit 的项目目录是当前 `app/`。业务代码的 Python 运行上下文是它的父目录，两者用途不同。

## 第一次使用

在 Codex 中打开当前 `app/` 目录并新建聊天，在技能列表中查找 `speckit`。若技能列表未更新，重新打开项目或重启 Codex。

以下指令在 Codex 聊天中逐条发送，不在终端执行。

先建立项目原则：

```text
$speckit-constitution
阅读 AGENTS.md 和现有相关实现，以当前架构、接口与验证方式为依据建立项目原则。
明确区分已经存在的约束和仍待确定的事项，不新增没有依据的业务规则。
```

再围绕一个真实需求开始：

```text
$speckit-specify 在这里描述本次要开发的功能、边界和期望结果。
```

不要为了使用工具而先为整个现有系统补写规格。每个新功能会在 `specs/` 中建立独立的规格目录，活动功能由 `.specify/feature.json` 记录。

## 常用流程

| 技能 | 用途 |
| --- | --- |
| `$speckit-constitution` | 建立或更新项目原则，通常每个项目先执行一次 |
| `$speckit-specify` | 记录本次功能需求、范围与验收条件 |
| `$speckit-clarify` | 按需澄清当前规格中的歧义 |
| `$speckit-plan` | 根据项目现状制定技术方案 |
| `$speckit-checklist` | 按需生成规格质量检查清单 |
| `$speckit-tasks` | 将方案拆为可执行任务 |
| `$speckit-analyze` | 检查规格、方案和任务是否一致 |
| `$speckit-implement` | 按任务实现并运行相应验证 |
| `$speckit-converge` | 对照规格与计划检查遗漏；有新增任务时继续实现再检查 |

规格与文档分析不能替代实际测试、运行检查和验收。

## 当前项目的运行条件

- 当前代码内部使用 `app.*` 导入，`PROJECT_ROOT` 指向当前目录的父目录。
- 接入时当前目录及父目录未发现依赖清单、`.env` 或父目录下的 `prompts/`；后续启动业务服务前，应根据实际部署环境补齐相应资源。
- `test/` 为真实流程演示脚本，不能直接视作不依赖外部服务的单元测试。
- 本次接入未生成业务功能规格，项目原则保留官方待填写模板。
- 初始化工具使用隔离环境，系统 `PATH` 中未新增全局 `specify` 命令；日常使用上述 Codex 项目技能即可。

## 官方文档

- [Spec Kit 官方仓库](https://github.com/github/spec-kit)
- [已有项目接入](https://github.github.com/spec-kit/how-to/existing-projects.html)
- [Codex 集成](https://github.github.com/spec-kit/integrations/codex.html)
- [规格驱动流程](https://github.github.com/spec-kit/reference/agentic-sdd.html)
- [Codex 本地技能](https://learn.chatgpt.com/docs/build-skills)
