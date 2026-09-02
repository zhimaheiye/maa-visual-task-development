---
name: maa-visual-task-development
description: >
  面向 MaaFramework 项目的 AI 辅助视觉自动化任务开发流程。用于把用户已经知道如何人工操作的 GUI 流程，交给本地/低成本执行 Agent 做实机勘察、截图、OCR、bbox/ROI 记录与重复测试，再由高能力模型设计状态机、处理复杂逻辑和疑难 Bug，最终固化为稳定、确定性的 MaaFramework Pipeline。
---

# MaaFramework 视觉任务开发 Skill

## 1. 目标

本 Skill 用于把“用户知道大致怎么人工操作”的流程，低摩擦地转化为稳定、可验证、确定性的 MaaFramework 自动化任务。

核心模式：

```text
用户定义业务语义
    ↓
执行 Agent 实机勘察
    ↓
生成 exploration 证据
    ↓
高能力模型设计状态机
    ↓
实现正式 Pipeline
    ↓
执行 Agent 本地验证
    ↓
失败证据回流并局部修正
    ↓
Candidate Ready
    ↓
用户实际使用验收
    ↓
用户接受后 commit / push
    ↓
Release 另行决定
```

最终运行时应尽量依赖 MaaFramework 的确定性能力，而不是长期依赖 LLM 临场操作。

---

## 2. 开始前先读项目规则

开始任何任务前，先查找并阅读项目自身约束，例如：

```text
AGENTS.md
PRODUCT.md
docs/handoff/CURRENT.md
README.md
已有 feature 文档
```

优先级：

```text
用户当前明确指令
>
项目根级规则
>
本 Skill
>
Agent 自身默认习惯
```

如果本 Skill 与项目已有架构冲突，不得强行套用本 Skill。

---

## 3. 三方分工与升级策略

### 用户

用户主要负责：

- 任务名称；
- 起点与终点；
- 人工操作规则；
- 分支策略；
- “什么算一次”；
- 循环参数的业务含义；
- 禁止操作；
- 最终实际使用验收。

用户不需要负责：

- 手工截图；
- 手工量 bbox / ROI；
- 每次 OCR 失败都亲自判断；
- 普通代码修改与回归；
- 在多个 Agent 之间持续充当人工路由器。

### 执行 Agent

执行 Agent 优先负责：

```text
连接模拟器
截图
OCR
点击 / 滑动
测量 bbox
推荐 ROI
记录状态转移
记录耗时
生成 exploration 包
Schema 校验
运行 Pipeline
保存失败截图与日志
本地代码修改
重复回归测试
```

探索阶段，执行 Agent 不是默认架构师，但也不应因为普通技术问题频繁停下来询问用户。

### 高能力模型

高能力模型优先负责：

- 状态机设计；
- 区分业务状态与视觉变体；
- OCR / TemplateMatch / FeatureMatch / ColorMatch 取舍；
- `next` / `on_error` / loop 设计；
- 跨节点计数与计时；
- 是否需要 Python Agent；
- 疑难 Bug 分析；
- 控制修改范围；
- 判断什么时候真的需要用户介入。

### Escalation Policy

默认把问题按性质路由：

```text
实机事实未知
→ 执行 Agent 自己截图 / OCR / 测试

技术判断未知
→ 截图 + 日志 + 当前节点交给高能力模型
→ 得到结论后继续本地修改和回归

业务语义未知
→ 高能力模型确认确实需要用户决策
→ 再询问用户

功能达到 Candidate Ready
→ 停下来让用户实际使用验收
```

例如：

```text
“这个 ROI 应该扩大多少？”
→ 技术问题，不问用户

“这次 OCR 为什么少了最后一个字？”
→ 实机 + 技术问题，Agent 采证据后交给高能力模型

“这个奖励到底应该选左边还是右边？”
→ 业务问题，需要用户

“功能本地已经跑通，可以交付了吗？”
→ Candidate Ready，需要用户亲自验收
```

**普通 Debug、ROI 调整、模板回测、代码报错和本地回归不应成为打断用户的停点。**

---

## 4. 先定义业务语义，再碰 Pipeline

必须先明确：

```text
任务名称
起始状态
结束状态
一次任务的定义
是否循环
循环参数的业务含义
失败时停止还是重试
绝对禁止操作
```

特别注意：“一次”不能靠截图猜。

例如：

```text
开贝壳次数 = 2
```

如果业务上“一次”定义为：

```text
点击「立即开始」
→ 完成整轮
→ 点击「太好了」
→ 重新回到初始页
```

那么只有完整闭环后才算：

```text
completed += 1
```

不能擅自解释成“点击继续两次”。

---

## 5. Start Contract

每个任务都应明确声明启动契约：

```text
允许从什么页面启动
启动前必须满足什么条件
任务负责哪些导航
任务明确不负责哪些导航
```

不要为了“更智能”无证据地把所有入口恢复逻辑都塞进核心状态机。

如果 V1 要求用户已经位于某个页面，就明确写出来。

---

## 6. 实机探索阶段

探索阶段不是正式开发。

目标是把用户脑中的流程转成：

```text
真实画面
+ 状态名称
+ 识别候选
+ 动作
+ 动作后的真实结果
+ 耗时
```

每个稳定状态遵循：

```text
等待稳定
↓
截图
↓
观察 / OCR
↓
记录 bbox / 推荐 ROI
↓
只执行一个动作
↓
重新观察
```

禁止“看一眼以后连续点三步”。

用户基于记忆提供的 UI 描述应视为探索提示，而不是已经验证的视觉事实。

详细探索规范见：

```text
references/exploration.md
```

---

## 7. 状态不是单张截图

强制区分：

```text
业务状态
vs
视觉 variant
```

错误模型：

```text
状态 E = E_001.png
```

正确模型：

```text
状态 E
├─ E_001_center
├─ E_002_share
└─ E_003_other_layout
```

如果多个界面变体业务处理相同，优先让一个节点覆盖它们，而不是每个布局建一个新节点。

还要特别区分状态的**作用域**：

```text
当前对象耗尽
≠ 当前轮次结束
≠ 整个任务结束
```

看到 `0 / exhausted / 无剩余` 时，必须先确认它属于当前好友、当前页面、当前轮次还是全局任务。

---

## 8. 识别策略原则

先问：

```text
这个状态最稳定、最直接决定下一步的可观察特征是什么？
```

而不是先问“我要截哪张图”。

### OCR 优先场景

稳定文字按钮优先 OCR。

如果 OCR 结果本身就是按钮，优先让框架点击识别结果，不要无必要地再硬编码固定坐标。

### OCR expected 不等于完整 UI 文案

PP-OCR 可能把完整文案拆成多个文本框。

例如：

```text
视觉文案：保留一个随机奖品
OCR： [保留] [一个随机奖品]
```

此时完整 `expected` 可能失败。

应优先选择：

> 在目标 ROI 内、单个 OCR box 中稳定存在、且足够唯一的最小业务语义片段。

例如：

```text
expected = 随机奖品
```

### 不要用弱 expected 掩盖错误 ROI

如果现场证据显示：

```text
全屏 OCR = 0(0点刷新体力)
ROI OCR   = 0(0点刷新体
```

而 expected 是：

```text
刷新体力
```

首先检查 ROI 是否截断了目标文本。

正确顺序：

```text
比较完整 bbox 与 ROI 边界
→ 修正 ROI
→ 保留稳定业务语义 expected
→ 回归正常样本与异常样本
```

不要优先把：

```text
刷新体力
```

削弱成：

```text
刷新
```

否则可能只是把“漏检”换成未来的“误报”。

### TemplateMatch

适合：

- 没有稳定文字的图标；
- OCR 长期不稳定；
- 固定视觉特征。

生成模板前应：

```text
从与运行时一致的标准化截图源取图
→ 确定 bbox
→ 裁图
→ 立即 TemplateMatch 回测
→ 记录 score
```

不要因为一次 OCR 失败就立即切模板。

---

## 9. bbox 与 ROI

必须区分：

```text
bbox
= 本次实际检测到的目标范围

recommended_roi
= 正式识别时建议搜索的范围
```

ROI 应覆盖：

- 已知视觉变体；
- 轻微位置偏移；
- 合理 UI 动画余量；
- OCR 文本框的完整边界。

但不要无证据无限扩大到全屏。

### ROI 边界截断检查

OCR 异常时必须检查：

```text
目标完整 bbox 的 left/top/right/bottom
vs
ROI 的 left/top/right/bottom
```

尤其关注目标 bbox 是否贴近 ROI 右边界或下边界。

典型故障：

```text
完整文字 bbox 右端 = x393
ROI 右端          = x320
→ ROI OCR 永远只能看到前半句
```

此类问题通常不是“识别模型不稳定”，而是输入图像在识别前就被裁坏了。

如果新 variant 导致漏检：

```text
先比较旧 bbox / 新 bbox / 旧 ROI
```

若业务状态相同，优先局部扩大已有 ROI，而不是新建专用状态。

---

## 10. 动画与移动目标

第一版不要为了省几百毫秒而过度设计。

如果动作后有动画，优先等待可识别的稳定后继状态。

对于会移动或点击后消失的目标，可以用高频抓帧做单点时序实验：

```text
点击前
+0.25s
+0.5s
+1.0s
```

验证：

```text
目标多久消失
是否会留在原地
什么时候可以安全重新识别
```

再据此设置 `post_delay`，而不是拍脑袋等待。

---

## 11. Exploration 包

推荐：

```text
dev/exploration/<task_name>/
├─ README.md
├─ flow.json
└─ screenshots/
   ├─ A_001.png
   ├─ B_001.png
   ├─ E_001_center.png
   ├─ E_002_share.png
   └─ UNKNOWN_001.png
```

默认视为本地开发材料：

```text
不进入 Release
不自动提交 Git
```

示例结构见：

```text
assets/flow.example.json
```

---

## 12. 未知状态与自动继续

出现计划外稳定界面时，先停止**不安全的连续操作**，但不要把“停止点击”误解成“停止整个开发流程并叫用户”。

默认流程：

```text
暂停不确定点击
↓
截图
↓
标记 UNKNOWN_xxx
↓
记录由哪个动作进入
↓
记录文字 / 按钮 / 页面特征
↓
判断是动画、新状态还是 Bug
↓
技术问题交给高能力模型
↓
继续本地修改 / 测试
```

只有真正需要业务决策时才询问用户。

禁止自行发明业务逻辑，也禁止因为普通技术未知频繁打断用户。

---

## 13. 状态机设计

探索完成后，先画状态图，再改代码。

状态节点应表达：

```text
“我确认当前是什么状态”
```

而不是：

```text
“我希望接下来发生什么”
```

复杂页面结束后，推荐使用路由节点观察多个稳定后继状态，而不是固定 sleep 后假定结果。

例如：

```text
Router
├─ 全局终态
├─ 模态 / 特殊弹窗
├─ 当前对象局部终态
├─ 循环上限
└─ 普通可操作目标
```

特殊状态应优先于通用可点击目标，避免背景元素穿透弹窗或终态。

---

## 14. 循环与计数

不要把 MaaFramework 节点 `repeat` 误用于“重复整条子流程”。

`repeat` 通常只是重复当前节点 action。

如果业务需要：

```text
完整 A → ... → A
循环 N 次
```

应建立明确的轮次计数逻辑。

计数点必须放在业务上真正完成一轮的位置。

如果项目已有 Python Agent，可以使用最小 Custom Recognition / Action 管理共享状态。

Recognition 尽量保持纯判断；状态变化由显式 Action 驱动。

同时，计数器只记录 Pipeline 真正知道的事件：

```text
已执行一次气泡点击
```

不要把它包装成无法证明的：

```text
已成功获得一个奖励
```

---

## 15. 开发测试策略：单点 → 短跑 → 长链路 → 用户验收

不要开发阶段一上来跑 100 / 200 次。

推荐四层测试：

### 第一层：单点因果实验

```text
只验证一个动作到底发生了什么
```

例如：点击一个孤立目标，确认收益、消失时延和状态变化。

### 第二层：短跑

```text
N = 1 / N = 2
或少量对象
```

验证核心循环、状态重置、分支和停止边界。

### 第三层：完整长链路 E2E

冻结主要设计，运行真实长路径以覆盖：

```text
低概率弹窗
长时间网络波动
深链路状态
累计状态串扰
```

但必须注意：

> **长链路 E2E 跑通，不等于每个 Recognition 已覆盖所有视觉位置 variant。**

某个分支在两个样本上成功，只能证明这两个样本成功；不能自动推出未来所有同类布局都可靠。

### 第四层：用户实际使用验收

Agent / GPT 自测通过后，状态只能叫：

```text
Candidate Ready
```

不能直接叫“正式完成”。

必须让用户在真实使用方式中跑一次，得到用户反馈后再决定是否交付。

---

## 16. 现场 Bug 处理

遇到失败时：

```text
先采样
再修改
```

禁止：

```text
一失败就乱加 fallback
一失败就扩大所有 timeout
一失败就改成固定坐标
一失败就同时放宽 ROI、expected、threshold，导致无法判断哪项修复有效
```

现场至少保留：

```text
失败节点
完整截图
OCR 原始结果
bbox
识别 ROI
最近日志摘要
预期状态
实际状态
关键参数
第一次错误动作的时间点
```

### 优先寻找可证伪的物理根因

例如 OCR 漏检时，先回答：

```text
完整画面 OCR 能否识别？
ROI 内 OCR 能否识别？
目标 bbox 是否超出 ROI？
文字是否被 OCR 分框？
页面是否尚未稳定？
```

如果已经有 smoking gun，就局部修复，不再泛化猜测。

### 修一个变量，再回归

典型顺序：

```text
确认 ROI 截断
→ 只修 ROI
→ 保留 expected
→ 耗尽样本回归
→ 正常样本反向回归
```

### 日志要能解释分支

对于重要跳过 / 终态分支，建议提供明确日志，例如：

```text
[好友摸宝] 当前好友体力已耗尽，跳过气泡采集
```

这样用户再次反馈异常时，可以区分：

```text
Recognition 根本没命中
vs
Recognition 命中了但后续状态机出错
```

如果本地截图路径对远端模型不可见：

> 执行 Agent 应把关键 PNG 直接作为聊天附件注入，并同时附失败节点、预期、实际和日志摘要。

详细 Bug 处理规则见：

```text
references/troubleshooting.md
```

---

## 17. Candidate Ready、用户验收与提交纪律

这是强制交付门槛。

状态必须明确分层：

```text
Implemented
→ Agent Verified
→ Candidate Ready
→ User Accepted
→ Committed / Pushed
→ Released（可选、单独决定）
```

### Agent Verified ≠ User Accepted

即使满足：

```text
Schema 通过
短跑通过
长链路 E2E 通过
0 卡死
若干样本 100% 命中
```

也不能替代用户在真实使用方式里的验收。

执行 Agent 和高能力模型认为功能已经可以交付时，才应停下来让用户介入。

### 用户验收前

默认允许：

```text
继续本地修改
继续 Debug
继续重复测试
继续技术闭环
```

默认不允许：

```text
把“自测通过”直接标记为正式完成
commit
push
release
```

除非用户明确提前授权这些动作。

### 用户接受带已知限制的版本也是有效完成

用户可能反馈：

```text
“仍有小概率失败，但比之前好多了，暂时不用优化。”
```

此时正确状态是：

```text
Known Limitation
+
User Accepted
```

不是：

```text
100% Fixed
```

也不是：

```text
必须继续优化到理论完美才能交付
```

文档应如实记录残余问题与用户当前接受阈值。

### Commit / Push 与 Release 分离

用户允许提交源码并不等于允许发布新版本。

```text
commit / push
≠ release
```

Release、打包、发版必须按项目规则或用户单独要求执行。

---

## 18. Git 安全

默认：

```text
允许本地修改
允许本地测试
不在 Candidate Ready 前自动提交最终功能
```

以下默认视为私人 / 临时开发材料，除非用户明确要求，不得提交：

```text
dev/exploration/
notes/
临时截图
测试日志
私人待办
待办.md
scratch/
```

禁止默认执行：

```text
git add .
git add -f <ignored-file>
```

提交前必须检查：

```bash
git status --short
git diff --cached --name-only
```

如果用户已明确接受当前 Candidate，并要求提交，则可以完成 commit / push；是否 Release 仍单独判断。

---

## 19. 完成标准

一个任务不能因为“Schema 通过”或“Agent 长跑通过”就视为正式完成。

### Candidate Ready 至少要求

```text
Schema 校验通过
关键状态实机命中
完整闭环成功
循环边界正确
任务重启状态正确
至少覆盖已知视觉 variants
没有已知危险误触
已知技术问题已处理到可供用户实际试用
```

### 正式交付还要求

```text
用户实际使用验收
用户明确接受当前效果或已知限制
```

只有之后才进入 commit / push；Release 另行决定。

出现新视觉变体时，优先补充 exploration 证据并局部修正，不推翻已经正确的主状态机。

---

## 20. 推荐阅读

本 Skill 的详细参考：

```text
references/exploration.md
references/troubleshooting.md
references/open-shell-case-study.md
references/friend-gem-case-study.md
assets/flow.example.json
```

两个真实案例分别强调：

```text
开贝壳
→ 状态机修正 / OCR 分框 / visual variants / 完整轮次计数

好友摸宝
→ 模糊需求 / 状态作用域 / 移动目标 / 安全 ROI / 长链路 E2E
→ ROI 边界截断根因 / Agent Verified ≠ User Accepted / Known Limitation
```
