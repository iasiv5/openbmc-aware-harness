# openbmc-aware-harness

面向 OpenBMC 固件开发的上下文基座。把 AI 协作中反复用到的规则、入口、设计文档、观察记录和任务落点放在独立仓库，让每轮会话从已有边界开始。

## 解决什么问题

1. AI 每轮会话自动读取规则文件，不依赖人每次重新交代上下文。
2. OpenBMC 协作中的层级归属、验证方式、沟通规范写成 `rules/` 下的文件，替代口头习惯和重复追问。
3. 设计文档进 `docs/specs/`，实施计划进 `docs/plans/`，观察记录进 `contexts/memory/`，临时实验进 `adhoc_jobs/`，不散落在会话记录里。
4. 通过 `rules/skills/` 按需引入新能力，出现明确需求时再扩展。

## 仓库结构

```raw
openbmc-aware-harness/
├── AGENTS.md                # AI 启动入口，定义每轮 session 读取链
├── CLAUDE.md                # Claude Code 入口
├── .github/
│   └── copilot-instructions.md  # GitHub Copilot 入口
├── rules/
│   ├── SOUL.md              # AI 身份与行为准则
│   ├── USER.md              # 服务对象画像
│   ├── WORKSPACE.md         # 目录路由表
│   ├── COMMUNICATION.md     # 沟通规范
│   ├── axioms/              # 从团队经历提炼的决策公理
│   └── skills/              # 可复用能力（工作流、最佳实践等）
├── docs/
│   ├── specs/               # 设计文档
│   └── plans/               # 实施计划
├── contexts/
│   └── memory/
│       └── OBSERVATIONS.md  # 共享观察与经验沉淀
├── adhoc_jobs/              # 一次性任务和实验
└── workspace/               # 工作空间
```

## Quick Start

1. 确认 `rules/` 下的 5 个核心文件存在：SOUL → USER → WORKSPACE → COMMUNICATION → skills/INDEX。开一个新 AI 会话，问"启动读取链是什么"，验证回答指向这 5 个文件。
2. 根据团队实际情况复核 USER.md（服务对象、常见任务）和 WORKSPACE.md（目录路由），确保反映当前状态。
3. 产出第一批工件：设计放 `docs/specs/`，计划放 `docs/plans/`，观察放 `contexts/memory/OBSERVATIONS.md`，实验放 `adhoc_jobs/`。
4. 接入真实工作流时，在计划文档中写明 OpenBMC 源码根目录和验证命令；只在出现明确需求时扩展 skills。

## 致谢

本项目受 [grapeot (Yan Wang / 鸭哥)](https://github.com/grapeot) 的 [context-infrastructure](https://github.com/grapeot/context-infrastructure) 项目启发并基于其架构思路构建。感谢鸭哥在 AI 上下文工程领域的开创性探索。