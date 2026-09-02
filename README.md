# maa-visual-task-development

一个面向 **MaaFramework 视觉自动化任务开发** 的 Agent Skill。

它解决的不是“教 AI MaaFramework 语法”本身，而是另一层问题：

> 当用户已经知道一个 GUI 流程应该怎么人工操作时，如何让本地/低成本执行 Agent 负责实机勘察、截图、OCR、bbox/ROI 记录与重复测试，再让高能力模型负责状态机设计、复杂逻辑和疑难 Bug，最终固化成稳定的 MaaFramework Pipeline。

## 适用场景

- 新增签到、领奖、领取、菜单操作、日常任务等视觉自动化流程；
- 用户知道人工步骤，但不想自己截图、量 ROI、写大量 Pipeline；
- 希望把便宜 Agent 用在“跑腿和测试”，把昂贵模型用在真正需要推理的部分；
- 开发过程中存在动画、分支、循环、OCR 分框、多视觉布局等情况。

## 仓库结构

```text
.
├─ SKILL.md
├─ README.md
├─ references/
│  ├─ exploration.md
│  ├─ troubleshooting.md
│  └─ open-shell-case-study.md
└─ assets/
   └─ flow.example.json
```

`SKILL.md` 是主入口；详细规则拆到 `references/`，减少主 Skill 的长度。

## 核心工作流

```text
用户定义业务语义
    ↓
执行 Agent 实机勘察
    ↓
生成 exploration 包
    ↓
高能力模型设计状态机
    ↓
实现确定性 Pipeline
    ↓
执行 Agent 实机验收
    ↓
失败证据回流并局部修正
```

## 与 MaaFramework 技术型 Skill 的关系

本 Skill 更关注 **“如何开发一个真实视觉任务”** 的协作协议，而不是完整复制 MaaFramework Pipeline 协议文档。

在实际使用中，它适合与 MaaFramework 官方文档、MaaMCP、MaaPipelineEditor，以及其他专门讲解 MaaFramework API / Pipeline Protocol 的 Skill 配合使用。

## CC Switch

这是一个单 Skill 仓库，因此自定义 Skill 源可以直接指向：

```text
Owner: zhimaheiye
Repository: maa-visual-task-development
Branch: main
Subdirectory: 留空 / 仓库根目录
```

## 当前状态

第一版基于真实的 MaaHappyFish“开贝壳”任务开发过程整理，已经纳入以下实机踩坑：

- 业务上的“一次任务”与 UI 点击次数不是同一个概念；
- 一个业务状态可以对应多个视觉 variant；
- OCR 完整按钮文案可能被 PP-OCR 拆成多个文本框；
- OCR `expected` 应选择单个文本框内稳定、唯一的最小语义片段；
- 现场 Bug 应先采样，再局部修正，不应先凭猜测增加 fallback；
- 本地截图路径对远端模型不可见时，应把现场图片直接注入聊天作为证据。

## License

暂未选择许可证。若准备正式对外开放复用，建议之后补充明确的开源许可证。
