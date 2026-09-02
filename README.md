# maa-visual-task-development

一个面向 **MaaFramework 视觉自动化任务开发** 的 Agent Skill。

它解决的不是“教 AI MaaFramework 语法”本身，而是另一层问题：

> 当用户已经知道一个 GUI 流程应该怎么人工操作时，如何让本地/低成本执行 Agent 负责实机勘察、截图、OCR、bbox/ROI 记录与重复测试，再让高能力模型负责状态机设计、复杂逻辑和疑难 Bug，最终固化成稳定的 MaaFramework Pipeline。

## 适用场景

- 新增签到、领奖、领取、菜单操作、日常任务等视觉自动化流程；
- 用户知道人工步骤，但不想自己截图、量 ROI、写大量 Pipeline；
- 用户只能提供基于记忆的业务描述，实际 UI 需要 Agent 逐步实机确认；
- 希望把便宜 Agent 用在“跑腿和测试”，把昂贵模型用在真正需要推理的部分；
- 开发过程中存在动画、分支、循环、OCR 分框、多视觉布局、移动目标或长链路低概率状态等情况。

## 仓库结构

```text
.
├─ SKILL.md
├─ README.md
├─ references/
│  ├─ exploration.md
│  ├─ troubleshooting.md
│  ├─ open-shell-case-study.md
│  └─ friend-gem-case-study.md
└─ assets/
   └─ flow.example.json
```

`SKILL.md` 是主入口；详细规则和真实案例拆到 `references/`，减少主 Skill 的长度。

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

## Completed Case Studies

### 1. MaaHappyFish「开贝壳」

见：[`references/open-shell-case-study.md`](references/open-shell-case-study.md)

主要验证：

- “一次任务”的业务语义不能靠截图猜；
- 动作后的真实状态转移必须实机确认；
- 一个业务状态可以存在多个 visual variants；
- PP-OCR 可能把完整按钮文案拆成多个文本框；
- OCR `expected` 应优先选择单个文本框内稳定且足够唯一的最小语义片段；
- 本地失败截图应直接注入远端对话，让高能力模型看到真实现场。

### 2. MaaHappyFish「好友摸宝」

见：[`references/friend-gem-case-study.md`](references/friend-gem-case-study.md)

稳定实现基线：`MaaHappyFish@867ddeb`。

这个案例进一步验证：

- 用户的模糊记忆适合作为探索提示，而不是 UI 真相；
- 显眼视觉元素不一定具有业务意义；
- UI 数字可能延迟刷新，不能直接拿同页静态显示判断动作是否成功；
- 局部 Exhausted 与全局 Done 必须区分作用域；
- 移动目标可以通过高速抓帧测量消失时延，再据此设置 `post_delay`；
- 安全 ROI 比无脑全屏 TemplateMatch 更重要；
- Recognition 尽量保持纯判断，状态变化由明确 Action 驱动；
- 实机验收应分层：单点实验 → 5 位短跑 → 200 水族箱 / 44.1 分钟完整 E2E；
- 长链路测试能捕获短跑无法覆盖的低概率系统弹窗；
- 已实机验证的分支与仅保留骨架的分支必须在文档中区分。

## 三方协作原则

推荐把问题按性质路由：

```text
实机事实未知
→ 执行 Agent 自己截图 / OCR / 测试

技术判断未知
→ 将截图、日志、节点信息交给高能力模型

业务语义未知
→ 高能力模型确认确实需要业务决策后，再询问用户
```

目标是减少用户充当多 Agent 之间人工路由器的负担，同时避免 AI 擅自发明业务规则。

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

当前版本已经基于两个真实 MaaHappyFish 功能完成端到端案例沉淀：

```text
开贝壳
→ 短闭环、多视觉 variant、OCR 分框、循环计数

好友摸宝
→ 模糊需求、逐步实机探索、移动目标、局部状态、长链路 E2E、长尾弹窗
```

这些规则来自实际 MaaFramework 资源开发过程，而不是只根据理论文档整理。

## License

暂未选择许可证。若准备正式对外开放复用，建议之后补充明确的开源许可证。
