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

`SKILL.md` 是主入口；详细规则和真实案例拆到 `references/`。

## 核心工作流

```text
用户定义业务语义
    ↓
执行 Agent 实机勘察
    ↓
高能力模型设计状态机
    ↓
实现确定性 Pipeline
    ↓
Agent 本地验证与技术闭环
    ↓
Candidate Ready
    ↓
用户实际使用验收
    ↓
用户接受后 commit / push
    ↓
Release 单独决定
```

一个关键原则：

```text
Agent Verified
≠ User Accepted
```

Agent 的短跑、长跑和 E2E 测试用于把功能推进到 **Candidate Ready**；最终是否接受，必须由用户在真实使用方式里亲自验收。

## 三方协作原则

推荐把问题按性质路由：

```text
实机事实未知
→ 执行 Agent 自己截图 / OCR / 测试

技术判断未知
→ 将截图、日志、节点信息交给高能力模型
→ 继续本地修改和回归

业务语义未知
→ 高能力模型确认确实需要业务决策后，再询问用户

功能达到 Candidate Ready
→ 停下来让用户实际使用验收
```

普通 ROI 调整、模板回测、代码报错和本地 Debug 不应频繁打断用户。

## Completed Case Studies

### 1. MaaHappyFish「开贝壳」

见：[`references/open-shell-case-study.md`](references/open-shell-case-study.md)

主要验证：

- “一次任务”的业务语义不能靠截图猜；
- 动作后的真实状态转移必须实机确认；
- 一个业务状态可以存在多个 visual variants；
- PP-OCR 可能把完整按钮文案拆成多个文本框；
- OCR `expected` 应选择单个文本框内稳定且足够唯一的最小语义片段；
- 本地失败截图应直接注入远端对话，让高能力模型看到真实现场。

### 2. MaaHappyFish「好友摸宝」

见：[`references/friend-gem-case-study.md`](references/friend-gem-case-study.md)

当前源码基线：`MaaHappyFish@03ccef8`。

这个案例进一步验证：

- 用户的模糊记忆适合作为探索提示，而不是 UI 真相；
- 局部 Exhausted 与全局 Done 必须区分作用域；
- 移动目标可以通过高速抓帧测量消失时延；
- 安全 ROI 比无脑全屏 TemplateMatch 更重要；
- Recognition 尽量保持纯判断，状态变化由明确 Action 驱动；
- 测试应分层：单点实验 → 短跑 → 200 水族箱 / 44.1 分钟完整 E2E → 用户实际使用验收；
- 长链路测试能捕获低概率系统弹窗，但不能保证每个 Recognition 已覆盖所有位置 variant；
- 用户验收发现过一个典型 ROI 边界截断 Bug：完整 OCR 可见 `0(0点刷新体力)`，旧 ROI 却只输入到 `0(0点刷新体`；
- 找到此类物理根因时，应优先修 ROI，而不是无证据把 `expected` 从 `刷新体力` 削弱成 `刷新`；
- Agent 自测通过不等于用户实际使用无问题；
- 用户可以接受带已知残余问题的版本，此时应记录 `Known Limitation + User Accepted`，而不是宣称 `100% Fixed`；
- commit / push 与 Release 必须分开处理。

## 当前 FriendGem 案例的最终交付状态

```text
主要 ROI 截断根因已修复
→ 用户实际使用反馈明显改善
→ 仍有小概率耗尽识别失败
→ 用户接受当前效果，暂不继续优化
→ 源码已推送
→ 暂未发布新 Release
```

这也是本 Skill 很重要的一条工程原则：

> 自动化开发的“完成”不是 AI 自己宣布 100% 成功，而是对当前证据、已知限制和用户接受阈值做准确描述。

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

当前版本已经基于两个真实 MaaHappyFish 功能完成案例沉淀：

```text
开贝壳
→ 短闭环、多视觉 variant、OCR 分框、循环计数

好友摸宝
→ 模糊需求、逐步实机探索、移动目标、局部状态、长链路 E2E、长尾弹窗
→ ROI 边界截断根因、用户验收门槛、Known Limitation
```

这些规则来自实际 MaaFramework 资源开发过程，而不是只根据理论文档整理。

## License

暂未选择许可证。若准备正式对外开放复用，建议之后补充明确的开源许可证。
